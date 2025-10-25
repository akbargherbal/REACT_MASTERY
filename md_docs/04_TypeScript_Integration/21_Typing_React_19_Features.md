# Chapter 21: Typing React 19 Features

## Typing the `use` Hook

## Learning Objective

Correctly type the data returned by the `use` hook when consuming promises and context, leveraging TypeScript's powerful inference capabilities.

## Why This Matters

The `use` hook is a fundamental new primitive in React 19 for reading resources like promises and context. Understanding how to type it is essential for working with data fetching in Server Components and creating more readable client components, ensuring that the data you "unwrap" from these resources is fully type-safe.

## Discovery Phase

The primary use case for the `use` hook is unwrapping the value from a promise. This is especially common in async Server Components for data fetching.

Let's see how TypeScript infers the type of data fetched from a simple API.

```jsx
import React, { use } from 'react';

// Define the shape of the data we expect from the API.
type Todo = {
  userId: number;
  id: number;
  title: string;
  completed: boolean;
};

// A function that returns a promise of our data.
// We explicitly type the return value as `Promise<Todo>`.
async function fetchTodo(id: number): Promise<Todo> {
  const res = await fetch(`https://jsonplaceholder.typicode.com/todos/${id}`);
  if (!res.ok) {
    throw new Error('Failed to fetch');
  }
  return res.json();
}

// This is an async Server Component.
async function TodoItem({ id }: { id: number }) {
  // We pass the promise directly to `use`.
  const todoPromise = fetchTodo(id);
  const todo = use(todoPromise);

  // Because `fetchTodo` returns `Promise<Todo>`, TypeScript infers
  // that `todo` is of type `Todo`. We get full autocompletion!
  // `todo` is NOT a promise here, it's the resolved value.

  return (
    <div>
      <h3>{todo.title}</h3>
      <p>Status: {todo.completed ? 'Done' : 'Pending'}</p>
    </div>
  );
}

export default function App() {
    return <TodoItem id={1} />;
}
```

**Rendered Output** (assuming the API call is successful):

```
<h3>delectus aut autem</h3>
<p>Status: Pending</p>
```

The magic is in the inference. Because our `fetchTodo` function has a clear return type of `Promise<Todo>`, TypeScript knows that when `use` successfully unwraps it, the resulting `todo` variable must be of type `Todo`. There's no need for manual type annotations around the `use` call itself.

This makes data fetching code in React 19 incredibly clean and fully type-safe.

## Deep Dive

### Typing `use` with Context

The `use` hook can also read context. This provides a cleaner syntax than `useContext`, especially when you need to read context conditionally.

```jsx
'use client';
import React, { createContext, use, useState } from 'react';

// Define the shape of our context value.
type Theme = 'light' | 'dark';
type ThemeContextType = {
  theme: Theme;
  toggleTheme: () => void;
};

// Create the context with a default value.
// It's good practice to type the context here.
const ThemeContext = createContext<ThemeContextType | null>(null);

function ThemeToggleButton() {
  // Read the context value with `use`.
  const context = use(ThemeContext);

  // TypeScript knows `context` is of type `ThemeContextType | null`.
  // We must check for null in case we're outside a provider.
  if (!context) {
    throw new Error('ThemeToggleButton must be used within a ThemeProvider');
  }

  // Inside this block, `context` is narrowed to `ThemeContextType`.
  return (
    <button onClick={context.toggleTheme}>
      Switch to {context.theme === 'light' ? 'dark' : 'light'} mode
    </button>
  );
}

// Provider component
export default function App() {
  const [theme, setTheme] = useState<Theme>('light');

  const toggleTheme = () => {
    setTheme(t => (t === 'light' ? 'dark' : 'light'));
  };

  const themeValue = { theme, toggleTheme };

  return (
    <ThemeContext.Provider value={themeValue}>
      <div style={{ padding: 20, background: theme === 'light' ? '#fff' : '#333', color: theme === 'light' ? '#000' : '#fff' }}>
        <p>Current theme: {theme}</p>
        <ThemeToggleButton />
      </div>
    </ThemeContext.Provider>
  );
}
```

Just like with `useContext`, the type safety of `use(ThemeContext)` depends on how you create the context. By providing a generic argument to `createContext<ThemeContextType | null>`, we give TypeScript all the information it needs. The `use` hook simply infers its return type from the context object you pass to it.

### Common Confusion: `use(promise)` vs. `await promise`

**You might think**: "In an async component, `const data = use(promise)` is the same as `const data = await promise`."

**Actually**: While they both yield the resolved value of the promise, they have different behaviors within React. `use` integrates with React's rendering lifecycle, particularly with Suspense. If the promise is not yet resolved, `use` can signal to React to show a Suspense fallback and re-render the component when the data is ready. `await` simply blocks the rendering of the component until the promise resolves.

**Why the confusion happens**: The syntax looks similar, and in the "happy path" where data is ready instantly, the result is the same.

**How to remember**: Use `await` for server-side logic _before_ you start rendering. Use `use` when you are fetching data _as part of_ the rendering process for a component.

### Production Perspective

- **Type Your Fetching Functions**: The key to type safety with `use(promise)` is to have strongly-typed data fetching functions. Always define the return type as `Promise<MyDataType>`. This creates a clear contract that `use` can infer from.
- **Error Handling**: `use` will throw the promise's rejection reason if it fails. This is designed to be caught by a parent `<ErrorBoundary>`. Your types should reflect the success case; error cases are handled by React's error boundary mechanism.
- **Conditional `use`**: A major advantage of `use` over `useContext` is that it can be called conditionally. This is perfectly type-safe.
  ```typescript
  function MyComponent({ useTheme }) {
    let theme = 'default';
    if (useTheme) {
      // `context` is only defined and typed within this block.
      const context = use(ThemeContext);
      if (context) theme = context.theme;
    }
    return <div>Theme is: {theme}</div>;
  }
  ```

## Type-Safe Actions

## Learning Objective

Define strongly-typed Server and Client Actions to handle form submissions and mutations with full type safety for both the state and the payload.

## Why This Matters

Actions are a central feature of React 19 for handling data mutations. Without types, it's easy to make mistakes, like misspelling a form field name or returning an incorrect error state. Typing your actions creates a robust, self-documenting contract between your forms and your mutation logic.

## Discovery Phase

An "Action" in React is just a function. To make it type-safe, we simply need to type its arguments and return value. An action function typically receives two arguments: the previous state of the form, and the form's data payload.

Let's define a simple Server Action to create a to-do item.

```javascript
'use server'; // This directive marks it as a Server Action

// Step 1: Define the shape of the state our action will manage.
// It can include an error message or a success message.
export type FormState = {
  message: string;
  error?: boolean;
};

// Step 2: Define the action function with types.
// `prevState`: The state from the previous submission.
// `formData`: The data submitted from the <form>.
export async function createTodoAction(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  const todoText = formData.get('todo');

  // Basic validation
  if (!todoText || typeof todoText !== 'string' || todoText.length < 3) {
    return { message: 'Todo must be at least 3 characters long.', error: true };
  }

  try {
    // In a real app, you would save this to a database.
    console.log(`Creating todo: ${todoText}`);
    // Simulate a delay
    await new Promise(res => setTimeout(res, 500));
    return { message: `Successfully added "${todoText}"` };
  } catch (e) {
    return { message: 'Failed to create todo.', error: true };
  }
}
```

This is the foundation of a type-safe action. Let's break down the signature:
`async function createTodoAction(prevState: FormState, formData: FormData): Promise<FormState>`

1.  **`prevState: FormState`**: The first argument is the state returned by the _previous_ run of this action. We've typed it with our custom `FormState` type.
2.  **`formData: FormData`**: The second argument is the payload. For native `<form>` submissions, this is a `FormData` object. `FormData` is a standard web API type, so we don't need to define it ourselves.
3.  **`Promise<FormState>`**: The function is `async`, so it returns a promise. The resolved value of the promise must match the `FormState` shape. This is the new state that will be available to the UI.

By defining these types, we've created a clear contract. We know exactly what kind of state to expect and what the function will return.

## Deep Dive

### Accessing Form Data

The `formData.get('fieldName')` method returns a `FormDataEntryValue`, which is a union type: `string | File | null`. This means you often need to perform type checking to ensure the value is what you expect.

```javascript
'use server';
import { z } from 'zod'; // A popular validation library

export type AddItemState = {
  message: string;
  errors?: Record<string, string[]>;
};

// Using a schema for validation is a best practice.
const itemSchema = z.object({
  name: z.string().min(3, { message: 'Name is too short' }),
  quantity: z.coerce.number().min(1, { message: 'Quantity must be at least 1' }),
});

export async function addItemAction(prevState: AddItemState, formData: FormData): Promise<AddItemState> {
  // `Object.fromEntries` converts FormData to a plain object.
  const rawData = Object.fromEntries(formData.entries());

  // Validate the data against the schema.
  const validationResult = itemSchema.safeParse(rawData);

  if (!validationResult.success) {
    // `flatten()` formats the errors into a useful shape.
    const errors = validationResult.error.flatten().fieldErrors;
    return {
      message: 'Validation failed.',
      errors,
    };
  }

  // At this point, `validationResult.data` is fully typed as { name: string, quantity: number }
  const { name, quantity } = validationResult.data;
  console.log(`Adding item: ${name} (x${quantity})`);

  return { message: `Added ${name}!` };
}
```

This more robust example introduces a common pattern:

1.  **Schema Definition**: We use a library like Zod to define the expected shape and validation rules for our form data. This is a single source of truth.
2.  **Parsing**: We parse the raw `FormData` against the schema.
3.  **Typed Data**: If parsing is successful, Zod gives us a fully typed data object. This is much safer than manually calling `formData.get()` and checking types for every field.
4.  **Structured Errors**: If parsing fails, Zod provides a structured error object that we can return as part of our state, making it easy to display field-specific error messages in the UI.

### Production Perspective

- **Use Schema Validation**: For any non-trivial form, use a validation library like Zod, Valibot, or ArkType. It co-locates your validation logic, provides type safety after parsing, and generates structured error messages.
- **Server Actions vs. Client Actions**: The typing pattern is identical for both. A Client Action is just a function defined in a `'use client'` component that is _not_ marked with `async`. Server Actions are `async` and live in `'use server'` files or are defined within Server Components.
- **Progressive Enhancement**: Type-safe actions work seamlessly with progressively enhanced forms. The `FormData` object is the standard way browsers submit forms, so your action will work even if JavaScript is disabled on the client. Your types ensure the server-side handling is robust.

## Typing `useActionState`

## Learning Objective

Type the `useActionState` hook and understand how TypeScript infers the form state and action dispatcher types from the provided action function.

## Why This Matters

`useActionState` (formerly `useFormState`) is the primary hook for managing form state with Actions. Its type inference is excellent, but understanding _how_ it works is key to building complex, type-safe forms that handle pending states, validation errors, and success messages correctly.

## Discovery Phase

Let's take the `createTodoAction` we defined in the previous section and wire it up in a client component using `useActionState`.

```jsx
'use client';
import React from 'react';
import { useActionState } from 'react';
import { createTodoAction, type FormState } from './actions'; // Assuming actions are in actions.ts

// The initial state must match the `FormState` type.
const initialState: FormState = {
  message: '',
};

function TodoForm() {
  // Step 1: Pass the action and initial state to the hook.
  const [state, formAction, isPending] = useActionState(createTodoAction, initialState);

  // Step 2: TypeScript infers the types for you!
  // `state` is inferred as `FormState`
  // `formAction` is inferred as a function that takes a `FormData` payload
  // `isPending` is inferred as `boolean`

  return (
    <form action={formAction}>
      <input type="text" name="todo" placeholder="Add a new todo" />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Adding...' : 'Add Todo'}
      </button>
      {state.message && (
        <p style={{ color: state.error ? 'red' : 'green' }}>
          {state.message}
        </p>
      )}
    </form>
  );
}

export default TodoForm;
```

**Interactive Behavior**:

1.  Type "Read a book" and click "Add Todo".
2.  The button shows "Adding..." for half a second.
3.  A green message appears: "Successfully added "Read a book"".
4.  Type "a" and click "Add Todo".
5.  The button shows "Adding...".
6.  A red message appears: "Todo must be at least 3 characters long."

This is the magic of `useActionState`'s type inference. Because our `createTodoAction` was typed as `(prevState: FormState, formData: FormData) => Promise<FormState>`, the hook knows:

- The first element it returns (`state`) must be of type `FormState`.
- The second element (`formAction`) is the function to pass to the `<form>`'s `action` prop.
- The third element (`isPending`) is a `boolean`.

You get complete type safety with almost no manual type annotations at the hook call site.

## Deep Dive

### The `useActionState` Type Signature

Let's look at a simplified type definition for `useActionState` to understand what's happening.

```typescript
function useActionState<State, Payload>(
  action: (state: State, payload: Payload) => Promise<State>,
  initialState: State,
): [state: State, dispatch: (payload: Payload) => void, isPending: boolean];
```

This is a generic function. When you call it, TypeScript performs these steps:

1.  It looks at the `action` function you provided (`createTodoAction`).
2.  It infers `State` from the type of the first argument and the return value (`FormState`).
3.  It infers `Payload` from the type of the second argument (`FormData`).
4.  It then uses these inferred `State` and `Payload` types to define the types of the array it returns.

This is why having a well-typed action function is the most critical step. The hook's type safety flows directly from it.

### Common Confusion: "Is the third argument (`isPending`) new?"

**You might think**: "I've seen `useFormState` before, but it only returned two values."

**Actually**: Yes, the `isPending` boolean is a new and welcome addition in React 19. `useActionState` is the new name for `useFormState`, and it now returns a tuple with three elements: `[state, formAction, isPending]`.

**Why the confusion happens**: The name change and the added return value are recent updates. Many older examples will show the two-argument version.

**How to remember**: `useActionState` gives you the **State**, the **Action** to dispatch, and tells you if it **isPending**. The name itself helps you remember what it provides.

### Production Perspective

- **Initial State Type**: Ensure your `initialState` perfectly matches the `State` type your action expects. Any mismatch will be caught by TypeScript.
- **Displaying Field-Specific Errors**: When your action's state includes a dictionary of field errors (as in our Zod example), you can easily map them to the UI.

```jsx

// Assuming `state.errors` is of type `Record<string, string[]>`
return (
  <form action={formAction}>
    <input name="name" />
    {state.errors?.name && <p>{state.errors.name[0]}</p>}

    <input name="quantity" type="number" />
    {state.errors?.quantity && <p>{state.errors.quantity[0]}</p>}

    <button type="submit">Submit</button>
  </form>
);
```

- **Resetting Form State**: `useActionState` does not automatically clear the form state message after a successful submission. A common pattern is to wrap the form in another component that can reset the state by changing the `key` prop on the form component, causing it to remount with its initial state.

## Typing `useFormStatus`

## Learning Objective

Utilize the `useFormStatus` hook to create responsive form components that react to pending states, and understand its built-in TypeScript types.

## Why This Matters

Often, you want to give the user feedback _while_ a form submission is in progress. You might want to disable a button, show a spinner, or change some text. The `useFormStatus` hook provides this information without requiring you to pass `isPending` state down through props. Its types are provided by React, making it simple to use correctly.

## Discovery Phase

The most important rule of `useFormStatus` is that it **must be used in a component that is a descendant of a `<form>` element**. It will not work in the same component that renders the `<form>`.

Let's create a dedicated `SubmitButton` component that uses this hook.

```jsx
"use client";
import React from "react";
import { useFormStatus } from "react-dom"; // Note: imported from 'react-dom'

// This component MUST be rendered inside a <form>
function SubmitButton() {
  // `useFormStatus` returns an object with the status of the parent form.
  const status = useFormStatus();

  // TypeScript knows the shape of `status` automatically.
  // `status.pending` is a boolean.
  // `status.data` is a FormData object if pending, otherwise null.
  // `status.method` is 'get' or 'post'.
  // `status.action` is the action function itself.

  return (
    <button type="submit" disabled={status.pending}>
      {status.pending ? "Saving..." : "Save"}
    </button>
  );
}

// --- USAGE ---
import { useActionState } from "react";
import { someServerAction } from "./actions";

export default function UserProfileForm() {
  const [state, formAction] = useActionState(someServerAction, null);

  return (
    <form action={formAction}>
      <label>
        Name: <input type="text" name="name" />
      </label>
      {/* We use our status-aware button here */}
      <SubmitButton />
    </form>
  );
}

// Dummy action for demonstration
async function someServerAction() {
  await new Promise((res) => setTimeout(res, 1000));
  return null;
}
```

**Interactive Behavior**:

1.  Click the "Save" button.
2.  The button immediately becomes disabled and its text changes to "Saving...".
3.  After a 1-second delay (from our dummy action), the form submission completes, and the button returns to its original "Save" text and enabled state.

This works without any prop drilling. The `SubmitButton` is "aware" of the state of the parent `<form>` thanks to the `useFormStatus` hook. The types for the `status` object are provided by `@types/react-dom`, so you get autocompletion and type safety for free.

## Deep Dive

### The `FormStatus` Type

The `useFormStatus` hook returns an object of type `FormStatus`. Let's look at its definition:

```typescript
interface FormStatus {
  pending: boolean;
  data: FormData | null;
  method: "get" | "post" | null;
  action: ((formData: FormData) => Promise<void>) | null;
}
```

- **`pending`**: The most used property. `true` if the form is submitting, `false` otherwise.
- **`data`**: A `FormData` object representing the data that is currently being submitted. This is useful for creating optimistic UI updates, which we'll see in the next section. When not pending, it's `null`.
- **`method`**: The HTTP method used for the submission.
- **`action`**: A reference to the action function that was called.

### Common Confusion: "Why is `useFormStatus` in `react-dom`?"

**You might think**: "Shouldn't all hooks be in the `react` package?"

**Actually**: `useFormStatus` is in `react-dom` because its functionality is intrinsically tied to the DOM (specifically, the `<form>` element). It's part of the glue between React's component model and the browser's native form behavior. Other DOM-specific hooks, like `useFormState` in older versions, also lived here.

**Why the confusion happens**: It's one of the few hooks imported from a different package, which is an easy detail to forget.

**How to remember**: If the hook is directly related to a native **DOM** element's behavior (like `<form>`), it's likely in `react-**dom**`.

### Production Perspective

- **Component Encapsulation**: The primary benefit of `useFormStatus` is that it allows you to create highly reusable, encapsulated components. You can build a design system with `SubmitButton`, `FormSpinner`, or `FormErrorMessage` components that automatically react to form state without needing any props passed to them.
- **User Experience**: Always use `useFormStatus` (or the `isPending` value from `useActionState`) to disable submit buttons during a mutation. This prevents accidental double submissions, which is a common source of bugs and user frustration.
- **Debugging**: If `useFormStatus` isn't working, the first thing to check is its location in the component tree. It _must_ be a child of the `<form>` it's tracking. A common mistake is to place it in the same component that renders the `<form>`, which will not work.

## Typing `useOptimistic`

## Learning Objective

Apply generics to the `useOptimistic` hook to safely implement optimistic UI updates, providing instant feedback to users.

## Why This Matters

In a world of fast user interfaces, waiting for a server response can feel slow. Optimistic updates—updating the UI _before_ the server confirms the change—make an application feel instantaneous. The `useOptimistic` hook is React's built-in tool for this, and typing it correctly is crucial for preventing inconsistencies between the optimistic state and the real state.

## Discovery Phase

Let's build a simple message list where new messages appear instantly, even though the "server" takes a moment to respond.

```jsx
'use client';
import React, { useOptimistic, useRef } from 'react';

// Define the shape of a single message
type Message = {
  text: string;
  sending?: boolean; // A flag for optimistic items
};

// A dummy server action
async function sendMessage(message: FormData): Promise<Message> {
  const text = message.get('message') as string;
  await new Promise(res => setTimeout(res, 1000)); // Simulate network delay
  return { text };
}

export default function MessageList({ initialMessages }: { initialMessages: Message[] }) {
  // Step 1: `useOptimistic` is a generic hook.
  // `OptimisticState` is the type of the state (`Message[]`).
  // `ActionPayload` is the type of the data passed to the update function (`string`).
  const [optimisticMessages, addOptimisticMessage] = useOptimistic<Message[], string>(
    initialMessages,
    // Step 2: Define the update function.
    // It takes the current state and the payload, and returns the new optimistic state.
    (currentState, newMessageText) => [
      ...currentState,
      { text: newMessageText, sending: true },
    ]
  );

  const formRef = useRef<HTMLFormElement>(null);

  const formAction = async (formData: FormData) => {
    const newMessageText = formData.get('message') as string;
    // Step 3: Call the optimistic update function with the payload.
    // This updates the UI instantly.
    addOptimisticMessage(newMessageText);

    // Reset the form
    formRef.current?.reset();

    // Now, call the actual server action.
    await sendMessage(formData);
    // Once the promise resolves, React will automatically revert the optimistic
    // state and re-render with the final state from the server (if different).
  };

  return (
    <div>
      <ul>
        {optimisticMessages.map((msg, index) => (
          <li key={index} style={{ opacity: msg.sending ? 0.5 : 1 }}>
            {msg.text} {msg.sending && <small>(Sending...)</small>}
          </li>
        ))}
      </ul>
      <form action={formAction} ref={formRef}>
        <input type="text" name="message" placeholder="Type a message..." />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

**Interactive Behavior**:

1.  The list shows initial messages.
2.  You type "Hello" into the input and click "Send".
3.  Instantly, "Hello (Sending...)" appears in the list with 50% opacity. The input field clears.
4.  One second later, the "(Sending...)" text disappears, and the opacity returns to 100% as React reconciles the optimistic state with the real state.

### Typing Breakdown

The signature is `useOptimistic<State, Action>(passthroughState, updateFn)`.

1.  **`State`**: The first generic argument is the type of the state itself. In our case, it's `Message[]`. This is the type of `optimisticMessages`.
2.  **`Action`**: The second generic argument is the type of the payload you'll pass to the updater function. We chose `string` because we only need the message text to create the optimistic state. This is the type expected by `addOptimisticMessage`.
3.  **`passthroughState`**: This is the "real" state, which React will use as the basis for updates. Here, it's `initialMessages`.
4.  **`updateFn`**: A function `(currentState: State, action: Action) => State`. TypeScript enforces this signature. It ensures that our updater takes the correct state and payload types and returns the new state in the correct shape.

## Deep Dive

### Common Confusion: "Do I need to manually revert the state?"

**You might think**: "After the server action finishes, I need to call another function to remove the `sending: true` flag."

**Actually**: No, React handles this automatically. `useOptimistic` is designed to be "fire-and-forget." When the component re-renders after the server action completes (because the `initialMessages` prop might change), React will automatically discard the optimistic state and use the new "real" state.

**Why the confusion happens**: In manual implementations of optimistic updates, you often have to manage the "revert" logic yourself.

**How to remember**: The `useOptimistic` hook is a time-traveler. It shows you a possible future, but when the present catches up (the server responds), it automatically snaps back to reality.

### Production Perspective

- **When to Use**: Optimistic updates are best for actions that are very likely to succeed. For high-stakes operations (like a payment), it's often better to show a pending state and wait for server confirmation.
- **Handling Errors**: If the server action throws an error, React will also discard the optimistic update and revert to the last known "real" state. You should handle this error with an Error Boundary or by returning an error state from your action to display a message to the user.
- **Matching Keys**: For lists, ensure your keys are stable. When React reconciles the optimistic and real state, it uses keys to match items. If an optimistic item has a temporary key, make sure the server response includes that key so React can seamlessly merge the states.

## Server Component Types

## Learning Objective

Understand the typing conventions for async Server Components, including how to type their props and return values.

## Why This Matters

Server Components are a new paradigm in React. They execute on the server and can perform asynchronous operations like database queries directly within the component. Typing them correctly ensures that your data fetching logic is sound and that the props you pass between Server Components are safe.

## Discovery Phase

At its core, a Server Component is an `async` function that returns JSX. The typing follows naturally from this definition.

```jsx
import React from 'react';
// Assume db is a typed database client, like Prisma or Drizzle
import { db } from './db';

// Define the shape of props for this Server Component
type UserProfileProps = {
  userId: number;
};

// The component is an `async` function.
// The return type is implicitly `Promise<React.ReactElement>`.
export default async function UserProfile({ userId }: UserProfileProps) {
  // You can `await` data fetching directly inside the component.
  const user = await db.user.findUnique({ where: { id: userId } });

  if (!user) {
    return <div>User not found.</div>;
  }

  // `user` is fully typed based on your database schema.
  // We can safely access properties like `user.name`.
  return (
    <div>
      <h1>{user.name}</h1>
      <p>Email: {user.email}</p>
    </div>
  );
}
```

This looks remarkably simple, and the typing is straightforward:

1.  **Props**: You type the props (`UserProfileProps`) just like any other component.
2.  **Function Signature**: The function is marked `async`.
3.  **Return Type**: Because it's an `async` function returning JSX, TypeScript correctly infers the return type as `Promise<React.ReactElement>`. You rarely need to write this out explicitly.

The key takeaway is that you don't need special TypeScript syntax for Server Components. You just type them as asynchronous functions.

## Deep Dive

### What You Can't Do in a Server Component

The type system won't stop you from trying to use client-side features, but it's important to know the rules. Server Components cannot use:

- Hooks like `useState`, `useEffect`, `useContext`, etc. They have no state and are not interactive.
- Browser-only APIs like `window` or `localStorage`.
- Event handlers like `onClick` or `onChange`. Interactivity is handled by Client Components.

An attempt to use `useState` would look like this:

```typescript
// THIS IS AN ERROR
export default async function MyServerComponent() {
  const [count, setCount] = useState(0); // Error: React Hooks can only be called in a Client Component.
  return <div>...</div>;
}
```

Frameworks like Next.js will give you a clear error if you try to do this.

### Composing Server Components

You can easily compose Server Components and `await` them, just like any other async function.

```jsx
import React from 'react';
import UserProfile from './UserProfile';
import UserPosts from './UserPosts'; // Another async Server Component

async function UserDashboard({ userId }: { userId: number }) {
  // You can render components in parallel using Promise.all
  // or await them sequentially.

  return (
    <div>
      {/* React can handle a promise that resolves to JSX */}
      <UserProfile userId={userId} />

      <h2>Posts</h2>
      {/* You can also await a component before rendering it */}
      {await <UserPosts userId={userId} />}
    </div>
  );
}
```

React's renderer is designed to handle promises as children. When it encounters a promise (like the one returned by `<UserProfile>`), it can stream the rest of the page and fill in the content of `UserProfile` once its data has been fetched and it has rendered.

### Production Perspective

- **Data Fetching Co-location**: The primary benefit of Server Components is co-locating your data fetching with your view logic. Your types should flow from your data source (e.g., a typed database client or an OpenAPI-generated API client) through your component to ensure end-to-end type safety.
- **Props must be Serializable**: When a Server Component renders a Client Component (which we'll cover next), the props you pass must be serializable. This means they can be converted to a string and sent over the network. You can pass primitives, plain objects, and arrays. You cannot pass functions, Dates, Maps, Sets, or other complex objects that aren't part of the JSON standard. TypeScript won't enforce this rule by default, but frameworks will warn you.

## Client Component Types

## Learning Objective

Type Client Components correctly, understanding the role of the `'use client'` directive and the serialization boundary for props passed from Server Components.

## Why This Matters

Client Components are the interactive building blocks of your application. They are essentially the same components we've been building for years, but in the new server-first paradigm, it's crucial to understand how they receive data from their server-side parents and how to type that boundary correctly.

## Discovery Phase

A Client Component is any component in a file that starts with the `'use client'` directive. Once you add this directive, everything in that file is considered part of the client bundle.

Let's create an interactive `Counter` button that receives its initial count from a Server Component.

```jsx
// FileName: CounterButton.tsx
'use client'; // This directive makes it a Client Component.

import React, { useState } from 'react';

// Props are typed just like any other component.
type CounterButtonProps = {
  initialCount: number;
};

export default function CounterButton({ initialCount }: CounterButtonProps) {
  // Now we can use hooks like `useState`!
  const [count, setCount] = useState(initialCount);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

Now, let's use this Client Component from a Server Component.

```jsx
// FileName: page.tsx (A Server Component)
import React from "react";
import CounterButton from "./CounterButton";

export default async function HomePage() {
  // Server-side logic to determine the initial count.
  const initialCount = Math.floor(Math.random() * 10);

  return (
    <div>
      <h1>Welcome</h1>
      <p>Here is an interactive button:</p>
      {/* We pass the server-calculated value as a prop to the Client Component. */}
      <CounterButton initialCount={initialCount} />
    </div>
  );
}
```

**Interactive Behavior**:
The page loads with a button showing a random number between 0 and 9 (e.g., "Count: 7"). Each time you click the button, the count increments.

The typing is exactly what we've learned in previous chapters. The only new concept is the `'use client'` directive. From a TypeScript perspective, there is no difference in how you type a Client Component versus a traditional React component. The key is understanding the boundary between server and client.

## Deep Dive

### The Serialization Boundary

When a Server Component renders a Client Component, the props are serialized (turned into a string, typically JSON) and sent from the server to the client. The client then "hydrates" the Client Component using these props.

This means you **cannot** pass props that are not serializable.

```jsx
// In a Server Component:
import ClientComponent from "./ClientComponent";

// ✅ OK: Primitives, plain objects, arrays
const serializableProps = {
  text: "hello",
  count: 123,
  items: [{ id: 1, name: "item" }],
};
// <ClientComponent data={serializableProps} />

// ❌ ERROR: Functions, Dates, Maps, Sets, etc.
const nonSerializableProps = {
  onClick: () => console.log("I will not be sent to the client"),
  date: new Date(),
};
// <ClientComponent data={nonSerializableProps} /> // This will cause a runtime error in Next.js/etc.
```

**How do you type this?**

TypeScript doesn't have a built-in way to enforce that a type is "JSON serializable." This is a limitation you need to be aware of. The best practice is to keep props passed from server to client simple.

If you need to pass complex data, like a `Date`, convert it to a serializable format on the server (e.g., an ISO string) and then parse it back into a `Date` object on the client.

```jsx
// Server Component
import ClientTime from './ClientTime';
export default function ServerPage() {
  const now = new Date();
  return <ClientTime dateString={now.toISOString()} />;
}

// ClientTime.tsx
'use client';
import { useMemo } from 'react';
export default function ClientTime({ dateString }: { dateString: string }) {
  const date = useMemo(() => new Date(dateString), [dateString]);
  return <p>Time is: {date.toLocaleTimeString()}</p>;
}
```

This pattern ensures that only serializable data crosses the server-client boundary, keeping your application robust.

### Production Perspective

- **Keep Server Components Lean**: Server Components are for fetching data and composing other components. They should not contain complex logic.
- **Keep Client Components Small**: Use the `'use client'` directive at the "leaves" of your component tree. Create small, interactive components and compose them within larger Server Components. This minimizes the amount of JavaScript sent to the browser.
- **Shared Types**: It's a great practice to have a shared `types.ts` file that can be imported by both your Server and Client Components. This ensures that both sides of the boundary agree on the shape of the data being passed.

## Typing Async Transitions

## Learning Objective

Type the `useTransition` hook to manage loading states for asynchronous actions without blocking the UI, ensuring type safety for the transition callback.

## Why This Matters

When an action triggers a slow network request or a heavy computation, you don't want the UI to freeze. Transitions allow you to keep the UI interactive while the async work happens in the background. `useTransition` is the hook that powers this, and its typing is simple but essential for correct usage.

## Discovery Phase

Let's build a simple UI that "searches" for a query. The search is simulated to be slow, and we'll use `useTransition` to show a pending indicator without the UI feeling sluggish.

```jsx
'use client';
import React, { useState, useTransition } from 'react';

// A fake search function that returns a promise
async function search(query: string): Promise<string[]> {
  await new Promise(res => setTimeout(res, 500));
  if (query === '') return [];
  return [`Result for "${query}" 1`, `Result for "${query}" 2`];
}

export default function SearchApp() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);

  // Step 1: Call `useTransition`.
  const [isPending, startTransition] = useTransition();

  // TypeScript infers the types:
  // `isPending` is a `boolean`.
  // `startTransition` is a function: `(callback: () => void) => void`.

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const newQuery = e.target.value;
    setQuery(newQuery);

    // Step 2: Wrap the slow/async state update in `startTransition`.
    startTransition(() => {
      // This update is marked as a "transition".
      // React will keep the UI interactive while it processes.
      search(newQuery).then(res => {
        setResults(res);
      });
    });
  };

  return (
    <div>
      <input type="text" value={query} onChange={handleChange} />
      <div style={{ opacity: isPending ? 0.5 : 1 }}>
        <h3>Results:</h3>
        {results.map((res, i) => <p key={i}>{res}</p>)}
        {isPending && <p>Searching...</p>}
      </div>
    </div>
  );
}
```

**Interactive Behavior**:

1.  You type "React" into the input field.
2.  The input field updates instantly. The UI does not freeze.
3.  The results section becomes semi-transparent, and "Searching..." appears.
4.  After 500ms, the results for "React" appear, the opacity returns to normal, and "Searching..." disappears.

The typing is very straightforward:

- `useTransition()` returns a tuple `[boolean, (callback: () => void) => void]`.
- `isPending` is a simple boolean you can use to show loading indicators.
- `startTransition` takes a callback function. This function must not have any arguments and should return `void`. Inside this callback, you trigger the state update that depends on the async operation.

## Deep Dive

### `useTransition` vs. Actions

React 19 Actions have built-in support for transitions. When you use `useActionState`, the `isPending` value it returns is powered by the same transition mechanism.

**When should you use `useTransition` manually?**
You should use `useTransition` when you have an async update that is _not_ part of a native `<form>` submission. Common use cases include:

- Live search results as a user types.
- Filtering or sorting a large list of data on the client.
- Fetching new data when a tab is clicked.

For any data mutation triggered by a `<form>`, prefer using an Action with `useActionState` or `useFormStatus`, as they provide the same pending state with less boilerplate.

### Common Confusion: "Can I `await` inside `startTransition`?"

**You might think**: "Since I'm doing async work, I should make the callback `async`."

```typescript
// THIS IS NOT THE INTENDED PATTERN
startTransition(async () => {
  const res = await search(newQuery);
  setResults(res);
});
```

**Actually**: While this might appear to work, it's not the recommended pattern. `startTransition` is designed to be synchronous. It marks any state updates that happen _synchronously within its callback_ as transitions. The correct pattern is to call the async function and then call your state setter inside the `.then()` block, as shown in the main example.

**Why the confusion happens**: It's natural to associate `async/await` with any asynchronous operation.

**How to remember**: `startTransition` doesn't wait for your promise. It just "tags" the state updates inside its callback. The async work happens, and when it's done, the `.then()` callback triggers the tagged state update.

### Production Perspective

- **Improving User Experience**: Transitions are a powerful tool for improving perceived performance. Use them for any interaction that might take more than 50-100ms to prevent the UI from feeling janky or unresponsive.
- **Combining with `useDeferredValue`**: `useTransition` is for when you're updating state. A related hook, `useDeferredValue`, is for when you're re-rendering a part of the UI based on a value that can change rapidly (like a search query). They often solve similar problems, but `useTransition` gives you an explicit `isPending` state, which is often more useful.
- **Typing the Callback**: The type `() => void` for the `startTransition` callback is strict. If your function takes arguments or has a meaningful return value, it won't be accepted. Ensure the function you pass is a simple, argument-less thunk that triggers the state update.

## Module Synthesis 📋

## Module Synthesis

In this chapter, we've applied our foundational TypeScript knowledge to the cutting-edge features of React 19. We've seen how these new APIs were designed with TypeScript in mind, offering a superior developer experience with powerful type inference that reduces boilerplate and enhances safety.

We began with the **`use` hook**, demystifying how it seamlessly infers return types from promises and context, leading to cleaner data-fetching and context consumption patterns. We then established the core pattern for **Type-Safe Actions**, creating a robust contract for our form mutations using `FormData` and libraries like Zod for validation.

Building on that, we saw how hooks like **`useActionState`** and **`useFormStatus`** leverage the types from our actions to provide fully-typed state and status variables with zero manual annotation. We tackled the most advanced new hook, **`useOptimistic`**, using generics to create instantaneous-feeling UIs while maintaining a clear and safe distinction between the optimistic and server-confirmed states.

We then navigated the new architectural paradigm of **Server and Client Components**. We learned to type async Server Components as simple async functions and understood the critical **serialization boundary** when passing props to interactive Client Components marked with `'use client'`. Finally, we typed **`useTransition`**, giving us a tool to keep our applications responsive during asynchronous state updates.

## Looking Ahead

You are now proficient in typing both the classic and the modern, React 19-specific parts of the library. You can build complex, interactive, and performant applications with a high degree of type safety.

In the next chapter, **Chapter 22: Advanced TypeScript Patterns in React**, we will push our skills even further. We'll move beyond the basics of typing components and hooks to explore advanced, professional-grade patterns. We'll learn how to build highly flexible and reusable components using discriminated unions for props, create polymorphic components with the "as prop" pattern, and leverage conditional types to build truly dynamic and powerful component APIs. This is where we transition from being proficient to being an expert in TypeScript with React.
