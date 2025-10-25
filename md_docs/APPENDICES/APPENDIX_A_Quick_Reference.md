# Chapter A: Appendix A: Quick Reference

## React 19 Hook API Reference

## Learning Objective

Quickly reference the syntax and purpose of essential React 19 hooks.

## Why This Matters

This cheat sheet serves as a fast, scannable reference during development, saving you from constantly switching to documentation. It's designed to be your go-to guide for the most frequently used hooks.

## Core Hooks Reference

A concise summary of hooks for state, effects, context, and refs.

### State Hooks

- **`useState<S>(initialState: S | () => S): [S, Dispatch<SetStateAction<S>>]`**
  - **Purpose**: Adds local state to a component.
  - **Returns**: A stateful value and a function to update it.
  - **Usage**: `const [count, setCount] = useState(0);`

- **`useReducer<R extends Reducer<any, any>, I>(reducer: R, initializerArg: I, initializer?: (arg: I) => ReducerState<R>): [ReducerState<R>, Dispatch<ReducerAction<R>>]`**
  - **Purpose**: Manages complex state logic using a reducer function.
  - **Returns**: The current state and a `dispatch` function.
  - **Usage**: `const [state, dispatch] = useReducer(reducer, initialState);`

### 🆕 The `use` Hook

- **`use<T>(promise: Promise<T>): T`**
  - **Purpose**: Reads the value of a promise. Suspends the component until the promise resolves.
  - **Usage**: `const userData = use(fetchUserData(userId));`

- **`use<T>(context: Context<T>): T`**
  - **Purpose**: Reads the value of a React context. Can be called conditionally.
  - **Usage**: `const theme = use(ThemeContext);`

### Effect Hooks

- **`useEffect(setup: () => (void | (() => void)), dependencies?: DependencyList)`**
  - **Purpose**: Synchronizes a component with an external system (e.g., data fetching, subscriptions).
  - **Usage**: `useEffect(() => { /* side effect */ return () => { /* cleanup */ }; }, [deps]);`

- **`useLayoutEffect(setup, dependencies?)`**
  - **Purpose**: Same as `useEffect`, but fires synchronously after all DOM mutations. Use for measuring DOM layout.
  - **Usage**: `useLayoutEffect(() => { /* measure DOM */ }, [deps]);`

### Ref Hooks

- **`useRef<T>(initialValue: T): MutableRefObject<T>`**
  - **Purpose**: Creates a mutable object that persists for the lifetime of the component. Commonly used for DOM element access or storing mutable values that don't trigger re-renders.
  - **Usage**: `const inputRef = useRef(null);`

- **`useImperativeHandle<T, R extends T>(ref: Ref<T> | undefined, createHandle: () => R, dependencies?: DependencyList)`**
  - **Purpose**: Customizes the instance value that is exposed to parent components when using `ref`.
  - **Usage**: Used inside a component that receives a `ref` prop.

### Performance Hooks (Pre-Compiler)

**Note**: With the React 19 Compiler, these hooks are often unnecessary. They are included here for reference when working on codebases that haven't fully adopted the compiler or for specific edge cases.

- **`useMemo<T>(create: () => T, dependencies?: DependencyList): T`**
  - **Purpose**: Memoizes the result of an expensive calculation.
  - **Usage**: `const expensiveValue = useMemo(() => compute(a, b), [a, b]);`

- **`useCallback<T extends Function>(callback: T, dependencies?: DependencyList): T`**
  - **Purpose**: Memoizes a callback function.
  - **Usage**: `const handleClick = useCallback(() => { /* handle click */ }, [deps]);`

- **`startTransition(scope: () => void)`**
  - **Purpose**: Marks state updates as non-urgent, allowing other updates to interrupt them. Improves UI responsiveness.
  - **Usage**: `startTransition(() => { setSearchQuery(input); });`

## Actions and Form API Reference

## Learning Objective

Quickly reference the syntax and purpose of React 19's new hooks for Actions and form state management.

## Why This Matters

Actions are a fundamental shift in how React handles data mutations and form submissions. This reference provides a quick lookup for the new hooks that power this system, enabling you to build robust, progressively enhanced forms.

## Action Hooks

These hooks are designed to work together to manage the lifecycle of a server or client action.

- **`useActionState<State, Payload>(action: (state: State, payload: Payload) => Promise<State>, initialState: State, permalink?: string): [state: State, dispatch: (payload: Payload) => void, isPending: boolean]`**
  - **Purpose**: Manages the state of an action, including pending states and returned data.
  - **Returns**: The action's current state, a `dispatch` function to invoke it, and a boolean `isPending` flag.
  - **Usage**: `const [state, formAction, isPending] = useActionState(updateUser, null);`

- **`useFormStatus(): { pending: boolean, data: FormData | null, method: string | null, action: ((formData: FormData) => void | Promise<void>) | null }`**
  - **Purpose**: Provides status information about the last form submission. Must be used in a component rendered inside a `<form>`.
  - **Returns**: An object containing the `pending` state and metadata of the form submission.
  - **Usage**: `const { pending, data } = useFormStatus();`

- **`useOptimistic<State, Action>(passthrough: State, reducer: (state: State, action: Action) => State): [optimisticState: State, addOptimistic: (action: Action) => void]`**
  - **Purpose**: Allows you to show a temporary, "optimistic" state while an async action is in progress. React automatically reverts to the real state once the action completes.
  - **Returns**: The optimistic state and a function to apply a temporary update.
  - **Usage**: `const [optimisticMessages, addOptimisticMessage] = useOptimistic(messages, reducer);`

## Form Element Integration

With Actions, native HTML elements become more powerful.

- **`<form action={myAction}>`**: The `action` prop on a form can now accept a function directly. When the form is submitted, this function is called with the form's `FormData`.
- **`<button formAction={myOtherAction}>`**: A button inside a form can override the form's primary action, allowing for multiple submit actions (e.g., "Save Draft" vs. "Publish").

```jsx
import { useActionState, useFormStatus } from "react";

function SubmitButton() {
  // useFormStatus gives status of the parent <form>
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? "Submitting..." : "Submit"}
    </button>
  );
}

export default function MyForm({ updateItemAction }) {
  // useActionState manages the action's lifecycle
  const [error, submitAction, isPending] = useActionState(
    async (previousState, formData) => {
      const result = await updateItemAction(formData);
      if (result.error) {
        return result.error;
      }
      return null;
    },
    null,
  );

  return (
    <form action={submitAction}>
      <input type="text" name="itemName" />
      <SubmitButton />
      {error && <p style={{ color: "red" }}>{error}</p>}
    </form>
  );
}
```

## TypeScript Utility Types Cheat Sheet

## Learning Objective

Quickly reference common TypeScript utility types used in React development.

## Why This Matters

TypeScript enhances React development by providing type safety. Utility types are powerful tools for transforming existing types, reducing boilerplate, and creating more flexible and maintainable component APIs.

## Common Utility Types

- **`Partial<T>`**
  - **Purpose**: Makes all properties of type `T` optional.
  - **Usage**: `function updateProfile(data: Partial<UserProfile>) { ... }`

- **`Required<T>`**
  - **Purpose**: Makes all properties of type `T` required.
  - **Usage**: `type CompleteConfig = Required<PartialConfig>;`

- **`Readonly<T>`**
  - **Purpose**: Makes all properties of type `T` read-only.
  - **Usage**: `function displaySettings(settings: Readonly<AppSettings>) { ... }`

- **`Pick<T, K>`**
  - **Purpose**: Creates a new type by picking a set of properties `K` from type `T`.
  - **Usage**: `type UserAvatarInfo = Pick<User, 'name' | 'avatarUrl'>;`

- **`Omit<T, K>`**
  - **Purpose**: Creates a new type by removing a set of properties `K` from type `T`.
  - **Usage**: `type ButtonProps = Omit<React.HTMLProps<HTMLButtonElement>, 'onClick'>;`

- **`Record<K, T>`**
  - **Purpose**: Creates an object type with keys of type `K` and values of type `T`.
  - **Usage**: `const featureFlags: Record<string, boolean> = { 'new-dashboard': true };`

- **`Extract<T, U>`**
  - **Purpose**: Extracts from `T` those types that are assignable to `U`. Useful with union types.
  - **Usage**: `type TextAlignment = Extract<'left' | 'right' | 'center' | 1, string>; // 'left' | 'right' | 'center'`

- **`Exclude<T, U>`**
  - **Purpose**: Excludes from `T` those types that are assignable to `U`.
  - **Usage**: `type NonNumericStatus = Exclude<'success' | 'error' | 200 | 404, number>; // 'success' | 'error'`

- **`ReturnType<T>`**
  - **Purpose**: Obtains the return type of a function type `T`.
  - **Usage**: `type MyFunctionReturn = ReturnType<() => string>; // string`

- **`Parameters<T>`**
  - **Purpose**: Obtains the parameter types of a function type `T` as an array.
  - **Usage**: `type MyFunctionParams = Parameters<(name: string, age: number) => void>; // [string, number]`

## Common Patterns Quick Guide

## Learning Objective

Quickly identify and recall common React component patterns and their primary use cases.

## Why This Matters

Recognizing and applying established patterns leads to more readable, maintainable, and scalable React applications. This guide helps you choose the right pattern for a given problem.

### Component Design Patterns

- **Conditional Rendering**
  - **What it is**: Showing different JSX based on conditions (`&&`, ternary operator, or `if` statements).
  - **When to use**: Displaying loading states, error messages, user-specific content, or toggling UI elements.

- **Lifting State Up**
  - **What it is**: Moving state from a child component to its nearest common ancestor.
  - **When to use**: When multiple child components need to share and react to the same state.

- **Component Composition**
  - **What it is**: Building complex components by combining smaller, single-purpose components. Often uses the `children` prop.
  - **When to use**: For creating reusable layouts and wrappers like `Card`, `Modal`, or `Sidebar`.

- **Custom Hooks**
  - **What it is**: Creating your own hooks (functions starting with `use`) to extract and reuse stateful logic.
  - **When to use**: To share logic between components, such as data fetching (`useData`), form handling (`useForm`), or connecting to a browser API (`useLocalStorage`).

- **Container/Presentational Pattern**
  - **What it is**: Separating components into "container" components that manage logic and state, and "presentational" components that just render UI based on props.
  - **When to use**: In larger applications to enforce separation of concerns. With hooks, this pattern is often replaced by custom hooks that provide the logic.

- **Compound Components**
  - **What it is**: A set of components that work together to manage a shared, implicit state. Example: `<Tabs>`, `<TabList>`, `<Tab>`, `<TabPanel>`.
  - **When to use**: For building complex, expressive UI components like accordions, tabs, or dropdown menus where children need to implicitly communicate with the parent.

- **Render Props**
  - **What it is**: A technique for sharing code between components using a prop whose value is a function that returns a React element.
  - **When to use**: For sharing logic that needs to be tightly coupled with the rendering of a component. Largely superseded by custom hooks, but still useful in some library APIs.

## Performance Checklist

## Learning Objective

Quickly review key areas for performance optimization in a React 19 application.

## Why This Matters

A performant application provides a better user experience. This checklist provides a starting point for diagnosing and fixing common performance bottlenecks.

### General Checklist

1.  **Trust the Compiler**:
    - [ ] Have you enabled the React 19 Compiler? It handles most memoization automatically.
    - [ ] Are you avoiding manual `useMemo` and `useCallback` unless the compiler indicates a need for it or for specific referential equality cases?

2.  **State Management**:
    - [ ] Is state located as close as possible to where it's used? (Avoid unnecessary global state).
    - [ ] Are you using transitions (`startTransition` or hooks like `useTransition`) for non-urgent UI updates to prevent blocking user input?

3.  **Rendering Large Lists**:
    - [ ] Are you using virtualization (windowing) for long lists or grids? (e.g., with TanStack Virtual).
    - [ ] Does every list item have a stable and unique `key` prop?

4.  **Component Structure**:
    - [ ] Can you split large, slow components into smaller ones? This allows React to re-render only the parts of the UI that have changed.
    - [ ] Are you passing down components as props (e.g., `children`) instead of rendering them inside a parent that re-renders often? This can prevent unnecessary re-renders of the child.

5.  **Data Fetching**:
    - [ ] Are you using Server Components for data fetching where possible to move work to the server?
    - [ ] Are you using `Suspense` to stream UI and avoid "all-or-nothing" loading states?
    - [ ] Are you preloading data, scripts, and styles where appropriate using React 19's new resource loading APIs?

6.  **Bundle Size**:
    - [ ] Are you using code splitting by route? (e.g., with `React.lazy` or a framework's router).
    - [ ] Have you analyzed your bundle with a tool like `vite-plugin-visualizer` or `webpack-bundle-analyzer` to identify large dependencies?
    - [ ] Are you tree-shaking libraries effectively?

7.  **Profiling**:
    - [ ] Have you used the React DevTools Profiler to identify which components are re-rendering unnecessarily and why?
    - [ ] Have you checked for long-running effects in `useEffect` or expensive calculations that could be optimized?

## Compiler Optimization Guide

## Learning Objective

Understand how to write React code that is optimized effectively by the React 19 Compiler.

## Why This Matters

The React 19 Compiler (React Forget) is a game-changer for performance, but it works best when you follow certain patterns. Writing compiler-friendly code ensures you get maximum performance benefits with minimal manual effort.

### Core Principles for Compiler-Friendly Code

The compiler is designed to understand standard JavaScript and React patterns. The goal is not to write "special" code for the compiler, but rather to write clear, predictable React code.

1.  **Follow the Rules of Hooks**:
    - The compiler relies on the static analyzability of hooks. Always call hooks at the top level of your component. Don't call them inside loops, conditions, or nested functions.

2.  **Keep State Mutations Local and Direct**:
    - **Do**: `setCount(count + 1)`
    - **Avoid**: Complex, indirect state updates where the compiler might lose track of dependencies. For example, avoid mutating state based on values derived from multiple, hard-to-trace function calls.

3.  **Write Components as Pure Functions (as much as possible)**:
    - Given the same props and state, your component should ideally render the same output. The compiler is designed to memoize components that behave like pure functions. Avoid side effects during the render phase.

4.  **Avoid Conditional Hook Calls**:
    - This is a core rule of hooks, but it's especially important for the compiler. The compiler needs to know exactly which hooks will run on every render.
    - **Do**: `const theme = use(ThemeContext); if (isMobile) { ... }`
    - **Don't**: `if (isMobile) { const theme = use(ThemeContext); }` (This is now possible with `use`, but can be harder for the compiler to optimize than a top-level call).

5.  **Limit Prop Drilling of Unstable Objects**:
    - The compiler is good at memoizing objects and functions created within a component. However, if you pass a new object literal or function through many layers of props, it can still cause re-renders in components that are not compiled.
    - **Good**: Let the compiler handle memoization at the source.
      ```jsx
      function Parent() {
        // The compiler will memoize this object and callback
        const style = { color: "blue" };
        const handleClick = () => console.log("clicked");
        return <Child style={style} onClick={handleClick} />;
      }
      ```

### When to Intervene Manually

You generally don't need `useMemo` or `useCallback` anymore. However, there are rare cases where you might still need them:

- **Passing functions to non-React systems**: If you're passing a callback to a third-party library that relies on stable function identity (e.g., `someLibrary.addEventListener('event', callback)`), you might need `useCallback`.
- **Fine-tuning dependencies for `useEffect`**: If an effect depends on a complex object or function, and you need precise control over when it re-runs, manual memoization might be necessary.
- **When the compiler tells you to**: In development mode, the compiler may provide warnings or suggestions. Follow its guidance.

**The Golden Rule**: Start without manual memoization. Profile your application. Only add `useMemo` or `useCallback` if you identify a specific, measurable performance bottleneck that the compiler is not solving.

## Module Synthesis 📋

## Appendix A: Synthesis

This appendix serves as your high-density reference for the most critical APIs, patterns, and checklists in modern React development.

- **API References (A.1, A.2)** provided a quick lookup for both foundational hooks and the new React 19 Actions API. Use these to verify signatures and purpose without leaving your editor for long.
- **TypeScript Utilities (A.3)** armed you with the type-transformation tools necessary for building robust and flexible component props.
- **Common Patterns (A.4)** gave you a mental map for structuring your components, helping you choose the right architecture for the job.
- **Performance and Compiler Guides (A.5, A.6)** offered actionable checklists to ensure your applications are fast and efficient, leveraging the full power of the React 19 Compiler.

Keep this appendix handy. The goal isn't to memorize everything but to know where to find a reliable, concise answer quickly. As you progress, you'll internalize these concepts, but a good quick reference is an invaluable tool for any developer.
