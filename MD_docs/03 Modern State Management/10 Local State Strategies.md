# Chapter 10: Local State Strategies

## When useState is enough (most of the time)

## The Default Choice: `useState`

In React, state is the memory of your component. It's the data that changes over time and causes your UI to re-render. The most fundamental tool for managing this memory is the `useState` hook. For the vast majority of components you build, `useState` is not just sufficient—it's the best tool for the job due to its simplicity and directness.

The core philosophy of `useState` is **co-location**: state is declared right next to the JSX that uses it. This makes components easy to understand, debug, and reuse.

Let's build the anchor example for this chapter: a `UserProfileForm`. This form will allow a user to edit their profile information. We will build upon and refactor this single component throughout the chapter to explore different state management strategies.

### Phase 1: Establish the Reference Implementation

Our goal is to create a form with several fields: name, email, bio, role, and a toggle for notifications.

**Project Structure**:

We'll work within a single page for simplicity.

```
src/
└── app/
    └── chapter-10/
        └── page.tsx   ← Our component will live here
```

Here is the initial, "naive" implementation using multiple `useState` hooks. This is our baseline—it's correct, functional, and often the right way to start.

```tsx
// src/app/chapter-10/page.tsx
'use client';

import { useState, FormEvent } from 'react';

export default function UserProfilePage() {
  const [name, setName] = useState('Jane Doe');
  const [email, setEmail] = useState('jane.doe@example.com');
  const [bio, setBio] = useState('A software developer specializing in React.');
  const [role, setRole] = useState<'user' | 'admin'>('user');
  const [notificationsEnabled, setNotificationsEnabled] = useState(true);

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    const formData = { name, email, bio, role, notificationsEnabled };
    console.log('Submitting:', formData);
    alert(`Form submitted! Check the console for data.`);
  };

  return (
    <div className="max-w-2xl mx-auto p-8 bg-gray-50 rounded-lg shadow-md">
      <h1 className="text-2xl font-bold mb-6">Edit User Profile</h1>
      <form onSubmit={handleSubmit} className="space-y-4">
        <div>
          <label htmlFor="name" className="block text-sm font-medium text-gray-700">Name</label>
          <input
            id="name"
            type="text"
            value={name}
            onChange={(e) => setName(e.target.value)}
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
          />
        </div>
        <div>
          <label htmlFor="email" className="block text-sm font-medium text-gray-700">Email</label>
          <input
            id="email"
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
          />
        </div>
        <div>
          <label htmlFor="bio" className="block text-sm font-medium text-gray-700">Bio</label>
          <textarea
            id="bio"
            value={bio}
            onChange={(e) => setBio(e.target.value)}
            rows={3}
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
          />
        </div>
        <div>
          <label htmlFor="role" className="block text-sm font-medium text-gray-700">Role</label>
          <select
            id="role"
            value={role}
            onChange={(e) => setRole(e.target.value as 'user' | 'admin')}
            className="mt-1 block w-full px-3 py-2 border border-gray-300 bg-white rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
          >
            <option value="user">User</option>
            <option value="admin">Admin</option>
          </select>
        </div>
        <div className="flex items-center">
          <input
            id="notifications"
            type="checkbox"
            checked={notificationsEnabled}
            onChange={(e) => setNotificationsEnabled(e.target.checked)}
            className="h-4 w-4 text-indigo-600 focus:ring-indigo-500 border-gray-300 rounded"
          />
          <label htmlFor="notifications" className="ml-2 block text-sm text-gray-900">
            Enable Notifications
          </label>
        </div>
        <button
          type="submit"
          className="w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
        >
          Save Profile
        </button>
      </form>
    </div>
  );
}
```

### Analysis of the `useState` Approach

This code works perfectly. Each piece of state is independent and managed by its own `useState` hook.

**What `useState` excels at:**

1.  **Simplicity and Readability**: `const [value, setValue] = useState(initialValue)` is arguably the most intuitive API in React. It clearly states: "Here is a piece of state, and here is the function to change it."
2.  **Independence**: When state variables are not related, managing them separately makes sense. Changing the user's `name` has no logical connection to their `notificationsEnabled` setting. `useState` reflects this separation.
3.  **Performance**: React is highly optimized for `useState`. When you call `setName`, React knows that only the components using the `name` variable *might* need to re-render.

**When to stick with `useState`:**

-   When managing simple, primitive values (strings, numbers, booleans).
-   When state updates are independent of each other.
-   When the logic to calculate the next state is simple (e.g., `setCount(count + 1)`).
-   In most components you write. **`useState` should be your default.**

### Limitation Preview

Our current form is excellent for simple field updates. But what happens when we need to implement more complex state transitions? For example, what if we need a "Reset Form" button that reverts all fields to their original values, or a "Load Data" button that populates the entire form from a mock API call?

With our current setup, we would need to call all five `set...` functions in each of those handlers. This can become repetitive and error-prone. This is the exact problem we will tackle in the next section.

## useReducer for complex state logic

## Managing Complexity with `useReducer`

As we previewed, our `useState`-based form is simple and effective, but it has a weakness: its state update logic is scattered. Each `onChange` handler contains its own logic. If we add more complex actions that affect multiple state variables at once, this decentralized approach becomes a liability.

### Iteration 1: The Problem of Interrelated State Updates

**Current state recap**: Our `UserProfileForm` uses five separate `useState` hooks to manage its fields. It works well for individual field changes.

**Current limitation**: There's no centralized place to manage state logic. Actions like "resetting the form" or "loading data" require calling multiple state setters, leading to code duplication and potential bugs if a new field is added but forgotten in one of the handlers.

**New scenario introduction**: Let's add two new buttons to our form:
1.  **Reset Form**: Reverts all fields to their initial state.
2.  **Load Sample Data**: Populates the form with data for a different user, simulating an API response.

Let's first try to implement this using our existing `useState` setup.

```tsx
// src/app/chapter-10/page.tsx (Conceptual 'before' state)

// ... (imports and component definition)
  const [name, setName] = useState('Jane Doe');
  // ... (other useState hooks)

  const initialData = {
    name: 'Jane Doe',
    email: 'jane.doe@example.com',
    bio: 'A software developer specializing in React.',
    role: 'user' as const,
    notificationsEnabled: true,
  };

  const handleReset = () => {
    setName(initialData.name);
    setEmail(initialData.email);
    setBio(initialData.bio);
    setRole(initialData.role);
    setNotificationsEnabled(initialData.notificationsEnabled);
  };

  const handleLoadSampleData = () => {
    const sampleData = {
      name: 'John Smith',
      email: 'john.smith@example.com',
      bio: 'Lead Engineer with a passion for TypeScript.',
      role: 'admin' as const,
      notificationsEnabled: false,
    };
    setName(sampleData.name);
    setEmail(sampleData.email);
    setBio(sampleData.bio);
    setRole(sampleData.role);
    setNotificationsEnabled(sampleData.notificationsEnabled);
  };

  // ... (rest of the component with new buttons)
  /*
  <div className="flex space-x-2 mt-4">
    <button type="button" onClick={handleLoadSampleData}>Load Sample Data</button>
    <button type="button" onClick={handleReset}>Reset</button>
  </div>
  */
```

### Diagnostic Analysis: Reading the Failure

This isn't a runtime failure, but a **maintainability and scalability failure**.

**Browser Behavior**:
The UI works exactly as expected. Clicking "Reset" resets the form. Clicking "Load Sample Data" loads the new data.

**Code Analysis**:
The problem lies in the code's structure.
- **Repetition**: The logic for updating the form state is duplicated. Both `handleReset` and `handleLoadSampleData` list out every single state setter.
- **Fragility**: If we add a new field, say `username`, we must remember to add `const [username, setUsername] = useState('')` AND update `handleReset` AND `handleLoadSampleData`. Forgetting one of these will introduce a subtle bug where resetting the form doesn't reset the new field.
- **Lack of Clarity**: The event handlers are cluttered with implementation details (`setName`, `setEmail`, etc.). They should ideally describe *what* the user is doing (e.g., "resetting the form"), not *how* the state is updated.

**Let's parse this evidence**:

1.  **What the user experiences**: A perfectly working form.
2.  **What the code reveals**: A maintenance problem waiting to happen. The state update logic is scattered and repetitive.
3.  **Root cause identified**: We've coupled the *intent* of an action (e.g., "reset form") with the low-level *mechanics* of updating each state slice.
4.  **Why the current approach can't solve this**: `useState` is designed for independent state slices. It doesn't provide a built-in mechanism for coordinating updates across multiple slices.
5.  **What we need**: A way to centralize all our state update logic into a single place, so that our event handlers can simply declare their intent by dispatching an "action".

### Technique Introduced: `useReducer`

The `useReducer` hook is React's built-in solution for complex state management. It's an alternative to `useState` that is better suited for managing state objects with multiple sub-values and handling complex, multi-part state transitions.

It works with three core concepts:
1.  **State**: A single object that holds all your component's state.
2.  **Reducer**: A pure function that takes the current `state` and an `action` object and returns the *new* state. It's where all your update logic lives. `(currentState, action) => newState`.
3.  **Dispatch**: A function you call from your event handlers to "dispatch" an action object to the reducer.

This pattern moves the "how to update" logic from your component's body into the centralized reducer function.

### Solution Implementation: Refactoring to `useReducer`

Let's refactor our `UserProfileForm`.

**Before** (Multiple `useState` hooks):
```tsx
const [name, setName] = useState('Jane Doe');
const [email, setEmail] = useState('jane.doe@example.com');
// ... and so on
```

**After** (A single `useReducer` hook):

First, we define the types for our state and actions, and the initial state object.

```typescript
// src/app/chapter-10/reducer.ts (can be in the same file or separate)

export type State = {
  name: string;
  email: string;
  bio: string;
  role: 'user' | 'admin';
  notificationsEnabled: boolean;
};

// Define the actions our reducer can handle
export type Action =
  | { type: 'UPDATE_FIELD'; field: keyof State; value: any }
  | { type: 'SET_FORM_DATA'; payload: State }
  | { type: 'RESET' };

export const initialState: State = {
  name: 'Jane Doe',
  email: 'jane.doe@example.com',
  bio: 'A software developer specializing in React.',
  role: 'user',
  notificationsEnabled: true,
};

// The reducer function contains all state update logic
export function formReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'UPDATE_FIELD':
      return {
        ...state,
        [action.field]: action.value,
      };
    case 'SET_FORM_DATA':
      return { ...action.payload };
    case 'RESET':
      return initialState;
    default:
      return state;
  }
}
```

Now, let's update the component to use this reducer.

```tsx
// src/app/chapter-10/page.tsx (Refactored with useReducer)
'use client';

import { useReducer, FormEvent } from 'react';
import { formReducer, initialState, State } from './reducer'; // Assuming reducer is in a separate file

export default function UserProfilePage() {
  const [state, dispatch] = useReducer(formReducer, initialState);

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    console.log('Submitting:', state);
    alert(`Form submitted! Check the console for data.`);
  };

  const handleFieldChange = (field: keyof State, value: any) => {
    dispatch({ type: 'UPDATE_FIELD', field, value });
  };

  const handleLoadSampleData = () => {
    const sampleData: State = {
      name: 'John Smith',
      email: 'john.smith@example.com',
      bio: 'Lead Engineer with a passion for TypeScript.',
      role: 'admin',
      notificationsEnabled: false,
    };
    dispatch({ type: 'SET_FORM_DATA', payload: sampleData });
  };

  const handleReset = () => {
    dispatch({ type: 'RESET' });
  };

  return (
    <div className="max-w-2xl mx-auto p-8 bg-gray-50 rounded-lg shadow-md">
      <h1 className="text-2xl font-bold mb-6">Edit User Profile</h1>
      <form onSubmit={handleSubmit} className="space-y-4">
        {/* Name Input */}
        <div>
          <label htmlFor="name" className="block text-sm font-medium text-gray-700">Name</label>
          <input
            id="name"
            type="text"
            value={state.name}
            onChange={(e) => handleFieldChange('name', e.target.value)}
            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
          />
        </div>
        {/* ... other form fields updated similarly ... */}
        {/* Example: Checkbox */}
        <div className="flex items-center">
          <input
            id="notifications"
            type="checkbox"
            checked={state.notificationsEnabled}
            onChange={(e) => handleFieldChange('notificationsEnabled', e.target.checked)}
            className="h-4 w-4 text-indigo-600 focus:ring-indigo-500 border-gray-300 rounded"
          />
          <label htmlFor="notifications" className="ml-2 block text-sm text-gray-900">
            Enable Notifications
          </label>
        </div>
        
        <div className="flex space-x-2 pt-4">
            <button type="button" onClick={handleLoadSampleData} className="flex-1 justify-center py-2 px-4 border border-gray-300 rounded-md shadow-sm text-sm font-medium text-gray-700 bg-white hover:bg-gray-50">Load Sample Data</button>
            <button type="button" onClick={handleReset} className="flex-1 justify-center py-2 px-4 border border-gray-300 rounded-md shadow-sm text-sm font-medium text-gray-700 bg-white hover:bg-gray-50">Reset</button>
        </div>

        <button
          type="submit"
          className="w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
        >
          Save Profile
        </button>
      </form>
    </div>
  );
}
```

### Verification and Improvement Analysis

**Verification**: The form's behavior is identical to the user. All functionality remains.

**Expected vs. Actual Improvement**:
- **Centralized Logic**: All state update logic is now inside `formReducer`. If we add a new field, we only need to update the `State` type and `initialState`. The `RESET` and `SET_FORM_DATA` actions will handle it automatically.
- **Declarative Handlers**: The event handlers (`handleReset`, `handleLoadSampleData`) are now much cleaner. They describe the *user's intent* by dispatching an action, rather than implementing the state update themselves.
- **Testability**: The `formReducer` is a pure function. You can export it and write unit tests for it completely independently of React, by simply passing in a state and an action and asserting the result.

### When to Apply This Solution: `useState` vs. `useReducer`

| Scenario | `useState` (Preferred) | `useReducer` (Consider) |
| :--- | :--- | :--- |
| **State Shape** | Primitive (string, boolean) or simple object/array. | Complex object with multiple nested values. |
| **State Updates** | Simple, independent updates. | Multiple sub-values are updated together. |
| **Update Logic** | Next state is easily calculated from the event. | Next state depends on the previous state in a complex way. |
| **Example** | A toggle switch's `on`/`off` state. | A multi-step form's state, a shopping cart. |

**Rule of thumb**: Start with `useState`. When you find yourself writing event handlers that call multiple `set...` functions, or when the logic for determining the next state becomes complex, it's a strong signal to refactor to `useReducer`.

### Limitation Preview

Our component is now robust and maintainable. However, it's a single, monolithic component. In a real application, we would likely break this form down into smaller, reusable components (`<TextInput>`, `<SelectInput>`, `<FormActions>`).

When we do that, how will the child components access the `state` and `dispatch` function from the parent? We could pass them down as props, but this can lead to a problem called "prop drilling." This is the challenge we'll solve next.

## Context API: use sparingly

## Sharing State with the Context API

Our `UserProfileForm` is now well-structured internally thanks to `useReducer`. But it's still one large component. Good React architecture favors composition—breaking down large components into smaller, focused ones.

### Iteration 2: The Problem of Prop Drilling

**Current state recap**: Our form uses `useReducer` to manage its state logic in a clean, centralized way.

**Current limitation**: The component is monolithic. If we break it into child components, we'll need a way to provide the form's `state` and `dispatch` function to them.

**New scenario introduction**: Let's refactor `UserProfileForm` into a parent component that orchestrates several smaller children:
-   `UserDetailsSection`: Contains the `name` and `email` inputs.
-   `UserBioSection`: Contains the `bio` textarea.
-   `FormActions`: Contains the "Save", "Reset", and "Load Data" buttons.

The state and dispatch function will still live in the top-level `UserProfilePage` component. Let's see what happens when we pass them down as props.

```tsx
// src/app/chapter-10/page-with-prop-drilling.tsx (Conceptual 'before' state)

// Assume UserDetailsSection, UserBioSection, etc. are defined as separate components
// that accept state and dispatch (or specific handlers) as props.

// Example Child Component
const UserDetailsSection = ({ state, handleFieldChange }) => {
  return (
    <>
      <div>
        <label>Name</label>
        <input
          value={state.name}
          onChange={(e) => handleFieldChange('name', e.target.value)}
        />
      </div>
      <div>
        <label>Email</label>
        <input
          value={state.email}
          onChange={(e) => handleFieldChange('email', e.target.value)}
        />
      </div>
    </>
  );
};

// Parent Component
export default function UserProfilePage() {
  const [state, dispatch] = useReducer(formReducer, initialState);

  const handleFieldChange = (field: keyof State, value: any) => {
    dispatch({ type: 'UPDATE_FIELD', field, value });
  };
  
  // ... other handlers that call dispatch

  return (
    <form>
      {/* 
        Here's the prop drilling. We pass down state and handlers.
        If UserDetailsSection had its own children that needed this data,
        it would have to pass them down again.
      */}
      <UserDetailsSection state={state} handleFieldChange={handleFieldChange} />
      <UserBioSection state={state} handleFieldChange={handleFieldChange} />
      {/* ... and so on for other components */}
    </form>
  );
}
```

### Diagnostic Analysis: Reading the Failure

Again, this is not a runtime error but an architectural one.

**Browser Behavior**:
The application works flawlessly.

**Code Analysis**:
- **Prop Drilling**: The `state` object and `handleFieldChange` function are passed down from `UserProfilePage` to each child. If we had another layer of nesting, say `<FormSection>` containing `<UserDetailsSection>`, then `<FormSection>` would have to accept those props just to pass them along, even if it doesn't use them itself.
- **Tight Coupling**: The child components are tightly coupled to the parent. Their prop signatures are dictated by the parent's state structure.
- **Refactoring Burden**: If we need to pass another piece of data from the parent to a deeply nested child, we have to add it to the prop list of every intermediate component.

**Let's parse this evidence**:

1.  **What the user experiences**: A working form.
2.  **What the code reveals**: A verbose and rigid component structure that is difficult to maintain and refactor.
3.  **Root cause identified**: State needed by deeply nested components is held high up in the tree, forcing intermediate components to act as tunnels for props.
4.  **What we need**: A mechanism to "teleport" data from a provider component to any consumer component deep in the tree, without passing it through every level.

### Technique Introduced: The Context API

React's Context API is designed to solve exactly this problem. It allows a parent component (`Provider`) to make data available to any component in its subtree (`Consumer`), no matter how deep, without explicit prop passing.

It consists of three main parts:
1.  `createContext()`: Creates a context object.
2.  `<MyContext.Provider value={...}>`: A component that provides a value to all its descendants.
3.  `useContext(MyContext)`: A hook that lets a component subscribe to the context and get its value.

### Solution Implementation: Refactoring to use Context

Let's create a `FormContext` and refactor our components.

**Before** (Prop drilling):
Child components received props: `<UserDetailsSection state={state} handleFieldChange={...} />`

**After** (Using Context):

```tsx
// src/app/chapter-10/FormContext.tsx
import { createContext, useContext, Dispatch } from 'react';
import { State, Action } from './reducer';

type FormContextType = {
  state: State;
  dispatch: Dispatch<Action>;
};

// Create the context with a default value (can be null or a mock)
export const FormContext = createContext<FormContextType | null>(null);

// Custom hook for easier consumption
export function useFormContext() {
  const context = useContext(FormContext);
  if (!context) {
    throw new Error('useFormContext must be used within a FormProvider');
  }
  return context;
}
```

```tsx
// src/app/chapter-10/page.tsx (Refactored with Context)
'use client';

import { useReducer, FormEvent } from 'react';
import { formReducer, initialState } from './reducer';
import { FormContext } from './FormContext';
import { UserDetailsSection } from './UserDetailsSection'; // Assume these are now separate components
import { FormActions } from './FormActions';

export default function UserProfilePage() {
  const [state, dispatch] = useReducer(formReducer, initialState);

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    console.log('Submitting:', state);
    alert(`Form submitted! Check the console for data.`);
  };

  return (
    <FormContext.Provider value={{ state, dispatch }}>
      <div className="max-w-2xl mx-auto p-8 bg-gray-50 rounded-lg shadow-md">
        <h1 className="text-2xl font-bold mb-6">Edit User Profile</h1>
        <form onSubmit={handleSubmit} className="space-y-4">
          <UserDetailsSection />
          {/* Other sections would go here */}
          <FormActions />
        </form>
      </div>
    </FormContext.Provider>
  );
}

// src/app/chapter-10/UserDetailsSection.tsx
import { useFormContext } from './FormContext';

export function UserDetailsSection() {
  const { state, dispatch } = useFormContext();

  const handleFieldChange = (field: keyof typeof state, value: any) => {
    dispatch({ type: 'UPDATE_FIELD', field, value });
  };

  return (
    <>
      <div>
        <label htmlFor="name">Name</label>
        <input
          id="name"
          type="text"
          value={state.name}
          onChange={(e) => handleFieldChange('name', e.target.value)}
          // ... className
        />
      </div>
      {/* ... email field ... */}
    </>
  );
}
```

### Verification and The Hidden Trap

The form still works perfectly. We've successfully eliminated prop drilling. The child components are now decoupled from their parents; they only need to know about the `FormContext`.

**But we've introduced a new, more insidious problem: performance.**

**New scenario**: Let's analyze what happens when the user types in the "Name" input field.

### Diagnostic Analysis: The Performance Failure

**Browser Behavior**:
The UI seems fine on this small form. On a larger, more complex application, it might feel sluggish.

**React DevTools Evidence**:
Using the React DevTools Profiler, we can record a render while typing a single character into the name field.

**React DevTools - Profiler Tab**:
-   Recorded render: Typing "s" into the name field.
-   Components that re-rendered:
    -   `UserProfilePage` (because its state changed)
    -   `UserDetailsSection` (consumes context, expected)
    -   `UserBioSection` (consumes context, **unexpected!**)
    -   `FormActions` (consumes context, **unexpected!**)
-   Reason for re-render for all children: "Context changed".

**Let's parse this evidence**:

1.  **What the user experiences**: A working form, possibly with subtle input lag on complex pages.
2.  **What DevTools reveals**: When we update `state.name`, every single component that consumes our `FormContext` re-renders.
3.  **Root cause identified**: When the `value` prop of a Context Provider changes, React has no choice but to re-render **all** components that consume that context. It doesn't know which part of the `value` object changed; it only knows the object reference is new.
4.  **Why this is a problem**: Our `FormActions` component only needs `dispatch` (which never changes) and our `UserBioSection` only needs `state.bio`. They are re-rendering needlessly on every keystroke in the `name` field. This is a classic performance bottleneck.

### When to Apply Context (and When to Avoid It)

Context is a powerful tool, but it is not a general-purpose state manager.

-   **What it optimizes for**: Avoiding prop drilling.
-   **What it sacrifices**: Render performance. All consumers re-render on any change.

**When to choose Context:**
-   For **low-frequency updates**: Theme (light/dark mode), user authentication status, language/locale settings. Data that changes rarely.
-   For passing down **stable values**: Functions like `dispatch` that have a stable identity, or configuration that is set once.

**When to avoid Context:**
-   For **high-frequency updates**: Form state, mouse coordinates, animation values. Anything that changes on every keystroke or frame.

This leads us to the final, crucial question: if `useState` is too simple, `useReducer` creates a monolith, and `Context` has performance issues, what is the right way to structure state in a complex application?

## Lifting state up vs. pushing it down

## The Fundamental Trade-off: State Placement

We've journeyed from simple `useState` to centralized `useReducer` to shared `Context`, and at each step, we solved one problem while revealing another. This journey highlights the central question of state management in React: **Where should state live?**

The answer involves a trade-off between two primary strategies: lifting state up and pushing state down.

### The Two Philosophies

#### 1. Lifting State Up

This is the strategy of placing state in the lowest common ancestor of all components that need to read or write it.

-   **Pros**:
    -   Creates a "single source of truth."
    -   Allows sibling components to share and react to the same state.
    -   Makes state changes predictable and easy to trace.
-   **Cons**:
    -   Can lead to prop drilling if not paired with Context.
    -   Can cause performance issues by forcing a large part of the component tree to re-render, even if only a small part of the state changed. Our `Context` example was an extreme case of this.

#### 2. Pushing State Down (Co-location)

This is the strategy of keeping state as close as possible to the component that uses it.

-   **Pros**:
    -   **Excellent performance**. State updates are localized, causing only the component that owns the state (and its children) to re-render.
    -   **Encapsulation**. Components manage their own state, making them more reusable and easier to reason about in isolation.
-   **Cons**:
    -   Makes it difficult for sibling components to communicate or share state. To do so, you are forced to lift the state up.

### Iteration 3: Finding the Right Balance

Our `Context`-based form suffered because we lifted *all* the form state up, even though each component only cared about a small slice of it. The solution is not to abandon lifting state, but to be more precise about what we lift.

**A Better Solution: Composition and Callbacks**

Let's refactor our form one last time. The parent `UserProfilePage` will still own the state using `useReducer` (because the state logic is complex and shared), but instead of providing the entire `state` object via Context, it will pass down only the necessary pieces as props. This is a more controlled version of "lifting state up."

**Before** (Context-based, poor performance):
```tsx
// Parent
<FormContext.Provider value={{ state, dispatch }}>
  <UserDetailsSection />
</FormContext.Provider>

// Child
const { state, dispatch } = useFormContext();
// ... uses state.name, state.email
```

**After** (Props-based, performant):

```tsx
// src/app/chapter-10/page.tsx (Final, balanced version)
'use client';

import { useReducer, FormEvent, useCallback } from 'react';
import { formReducer, initialState, State } from './reducer';

// Child components are now "controlled components"
// They receive values and callbacks to report changes.

function UserDetailsSection({ name, email, onFieldChange }) {
  console.log('Rendering UserDetailsSection');
  return (
    <>
      <div>
        <label>Name</label>
        <input value={name} onChange={(e) => onFieldChange('name', e.target.value)} />
      </div>
      <div>
        <label>Email</label>
        <input value={email} onChange={(e) => onFieldChange('email', e.target.value)} />
      </div>
    </>
  );
}

function UserBioSection({ bio, onFieldChange }) {
  console.log('Rendering UserBioSection');
  return (
    <div>
      <label>Bio</label>
      <textarea value={bio} onChange={(e) => onFieldChange('bio', e.target.value)} />
    </div>
  );
}

export default function UserProfilePage() {
  const [state, dispatch] = useReducer(formReducer, initialState);

  // Use useCallback to ensure the function identity is stable between renders,
  // preventing unnecessary re-renders of children if we were to use React.memo.
  const handleFieldChange = useCallback((field: keyof State, value: any) => {
    dispatch({ type: 'UPDATE_FIELD', field, value });
  }, []); // dispatch is stable

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    console.log('Submitting:', state);
  };

  return (
    <div className="max-w-2xl mx-auto p-8">
      <h1 className="text-2xl font-bold mb-6">Edit User Profile</h1>
      <form onSubmit={handleSubmit} className="space-y-4">
        <UserDetailsSection 
          name={state.name} 
          email={state.email} 
          onFieldChange={handleFieldChange} 
        />
        <UserBioSection 
          bio={state.bio} 
          onFieldChange={handleFieldChange} 
        />
        {/* ... other components and buttons */}
      </form>
    </div>
  );
}
```

### Diagnostic Analysis: The Performance Win

If you run this version with the `console.log` statements and type in the "Name" input, you will see this in the console:

```
Rendering UserDetailsSection
Rendering UserBioSection
Rendering UserDetailsSection
Rendering UserBioSection
...
```

Wait, this is the same behavior as before! Both components re-render. Why? Because when the parent's state changes, React re-renders the parent, which in turn re-renders all its children by default.

To fix this, we need one final piece: `React.memo`. `memo` is a higher-order component that prevents a component from re-rendering if its props haven't changed.

Let's apply it:

```tsx
import { memo } from 'react';

const MemoizedUserDetailsSection = memo(UserDetailsSection);
const MemoizedUserBioSection = memo(UserBioSection);

// In UserProfilePage's return:
<MemoizedUserDetailsSection 
  name={state.name} 
  email={state.email} 
  onFieldChange={handleFieldChange} 
/>
<MemoizedUserBioSection 
  bio={state.bio} 
  onFieldChange={handleFieldChange} 
/>
```

Now, when you type in the "Name" input, the console output will be:

```
Rendering UserDetailsSection
Rendering UserDetailsSection
Rendering UserDetailsSection
...
```

`UserBioSection` no longer re-renders! We have achieved optimal performance. We lifted the state to the parent but passed down only the necessary props, allowing `memo` to prevent unnecessary re-renders of sibling components.

### Decision Framework: Where Should State Live?

Use this flowchart to decide on your state management strategy.

1.  **Does this state belong to a single component?**
    *   **Yes** -> Keep the state inside that component.
        *   Is the state logic complex or are multiple values updated together?
            *   **Yes** -> Use `useReducer`.
            *   **No** -> Use `useState`. (This is the most common case).

2.  **No, the state needs to be shared between components.**
    *   Are the components siblings?
        *   **Yes** -> **Lift state up** to their common parent. Pass data down as props and updates up via callbacks. Use `React.memo` on children to optimize performance.
    *   Are the components far apart in the tree?
        *   Does the state change frequently (e.g., form input)?
            *   **Yes** -> **Avoid Context**. Lift state up as described above, or consider a dedicated state management library (covered in Chapter 11).
        *   Does the state change infrequently (e.g., theme, user auth)?
            *   **Yes** -> **Context API** is an excellent choice to avoid prop drilling.

### The Journey: From Problem to Solution

| Iteration | Problem / Failure Mode | Technique Applied | Result | Performance Impact |
| :--- | :--- | :--- | :--- | :--- |
| 0 | Initial requirement | Multiple `useState` | Works, but scattered logic. | Baseline |
| 1 | Repetitive, error-prone update logic | `useReducer` | Centralized, robust logic. | Neutral |
| 2 | Prop drilling in component breakdown | `Context API` | Decoupled components. | **Negative**: Excessive re-renders. |
| 3 | Excessive re-renders from Context | Lift State Up + Props + `memo` | Decoupled & performant. | **Optimal** |

### Lessons Learned

-   **Start simple**: Always begin with `useState` and co-located state. It's often all you need.
-   **Centralize when complex**: Move to `useReducer` when state transitions become interdependent and complex.
-   **Use Context for static data**: Context is for avoiding prop drilling of data that rarely changes. It is not a substitute for a state manager.
-   **Lift state deliberately**: When sharing state between components, lift it to a common ancestor, but pass down only the minimal props required. Combine this with `React.memo` for performance.

Mastering local state is about understanding these trade-offs and choosing the simplest, most performant tool for the job at hand.
