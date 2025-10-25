# Chapter 20: Typing React 19 Components

## Function Component Types

## Learning Objective

Understand the modern, standard way to type a React function component and its props, and why it's preferred over legacy patterns like `React.FC`.

## Why This Matters

How you type your components sets the foundation for your entire application's type safety. Using the modern, explicit approach makes your components' APIs clearer, prevents common bugs related to implicit props like `children`, and aligns with the direction of the React ecosystem.

## Discovery Phase

Let's start with the simplest possible typed component. It takes no props and just renders a message.

```jsx
import React from 'react';

// A simple component with no props.
// The return type is implicitly inferred as JSX.Element or similar.
function WelcomeMessage() {
  return <h1>Welcome to Typed React!</h1>;
}

// You can be explicit about the return type if you want, but it's often not necessary.
function ExplicitWelcomeMessage(): React.ReactElement {
    return <h1>Welcome to Typed React!</h1>;
}
```

TypeScript's inference is excellent here. It knows that a function returning JSX is a valid React component. You generally don't need to annotate the return type.

The real typing happens when we introduce props. The standard way is to define a type for the props object and apply it to the function's first argument.

```jsx
import React from 'react';

// 1. Define the shape of the props object using a `type` alias.
type GreetingProps = {
  name: string;
};

// 2. Apply the type to the `props` argument.
// We use destructuring to pull out the `name` property.
function Greeting({ name }: GreetingProps) {
  return <h1>Hello, {name}!</h1>;
}

export default function App() {
  return <Greeting name="Alice" />;
  // Try passing a number: <Greeting name={123} />
  // TypeScript will immediately show an error.
}
```

**Rendered Output**:

```
Hello, Alice!
```

This is the pattern you will use for 95% of your components. It is simple, explicit, and easy to read.

## Deep Dive

### The Legacy `React.FC` Pattern

You will encounter another pattern in older codebases or tutorials: `React.FC` (or its full name, `React.FunctionComponent`).

Let's see the same `Greeting` component written with `React.FC`.

```jsx
import React from 'react';

type GreetingProps = {
  name: string;
};

// Legacy pattern using React.FC
const GreetingFC: React.FC<GreetingProps> = ({ name }) => {
  return <h1>Hello, {name}!</h1>;
};

// Modern, preferred pattern
function GreetingModern({ name }: GreetingProps) {
  return <h1>Hello, {name}!</h1>;
}
```

While they look similar, `React.FC` has two main drawbacks that have made it fall out of favor:

1.  **Implicit `children`**: As we discussed in Chapter 19, older versions of `React.FC` automatically added `children?: React.ReactNode` to your props. This meant you could pass children to a component that wasn't designed to use them, which was a source of bugs. While this has been fixed in the latest `@types/react`, the historical confusion remains.
2.  **Doesn't work well with generics**: Typing generic components (which we'll cover later in this chapter) is more cumbersome with `React.FC`.

The modern approach of typing props directly on the function argument is more direct and avoids these issues entirely.

### Common Confusion: `React.FC` vs. Plain Function

**You might think**: "`React.FC` seems more official. Isn't it better to use it to signal that this is a React component?"

**Actually**: A function that accepts a props object and returns JSX _is_ a React component. You don't need a special type to "bless" it as one. The modern pattern is more aligned with standard TypeScript function typing and is less verbose.

**Why the confusion happens**: `React.FC` was promoted in early TypeScript-React documentation, so it's widespread in older content. The community consensus has since shifted.

**How to remember**: Type your component like any other function. The props are the arguments, and the JSX is the return value. Keep it simple.

### Production Perspective

- **Consistency is Key**: Choose the modern pattern (`function MyComponent(props: MyProps)`) and stick with it across your project. Use a linter to enforce this consistency.
- **Readability**: The modern pattern is easier to read because the props type is directly next to the props it's describing.
- **Refactoring**: When refactoring an older codebase, consider converting `React.FC` components to the modern function syntax to improve consistency and explicitness, especially regarding the `children` prop.

## Typing Props

## Learning Objective

Define and apply structured types for component props using `type` or `interface`, including destructuring for cleaner access.

## Why This Matters

Props are the public API of your components. A well-defined props type acts as a contract, ensuring that components are used correctly. This is the single most important typing skill in React, as it prevents the most common category of bugs: passing incorrect data.

## Discovery Phase

Let's build a `UserProfile` card component that displays a user's information. We'll start by defining the shape of its props.

```jsx
import React from 'react';

// Step 1: Define the props type.
// Using `type` is a great default, as we discussed in Chapter 19.
type UserProfileProps = {
  username: string;
  avatarUrl: string;
  isActive: boolean;
  theme: 'light' | 'dark'; // Using a union type for specific options
};

// Step 2: Apply the type and destructure the props.
function UserProfile({ username, avatarUrl, isActive, theme }: UserProfileProps) {
  const cardStyle = {
    border: '1px solid #ccc',
    borderRadius: '8px',
    padding: '16px',
    maxWidth: '250px',
    backgroundColor: theme === 'dark' ? '#333' : '#fff',
    color: theme === 'dark' ? '#fff' : '#333',
  };

  const statusStyle = {
    color: isActive ? 'green' : 'grey',
  };

  return (
    <div style={cardStyle}>
      <img src={avatarUrl} alt={`${username}'s avatar`} width="80" style={{ borderRadius: '50%' }} />
      <h2>{username}</h2>
      <p style={statusStyle}>{isActive ? 'Online' : 'Offline'}</p>
    </div>
  );
}

export default function App() {
  return (
    <UserProfile
      username="JaneDoe"
      avatarUrl="https://i.pravatar.cc/80"
      isActive={true}
      theme="dark"
    />
  );
}
```

**Rendered Output**:

```
(A dark-themed card with an avatar, the username "JaneDoe", and the status "Online" in green text.)
```

This example demonstrates several key practices:

1.  **Descriptive Type Name**: `UserProfileProps` clearly indicates its purpose.
2.  **Primitive Types**: We use `string` and `boolean`.
3.  **Union Type**: The `theme` prop is restricted to either `'light'` or `'dark'`, preventing invalid values.
4.  **Destructuring**: In the function signature `({ username, ... }: UserProfileProps)`, we immediately pull the properties we need into local constants, making the component body cleaner.

## Deep Dive

### `interface` vs. `type` for Props

As covered in Chapter 19, you can also use an `interface` to define props.

```jsx
import React from 'react';

// Using `interface`
interface UserProfilePropsInterface {
  username: string;
  avatarUrl: string;
  isActive: boolean;
}

function UserProfileWithInterface({ username, avatarUrl, isActive }: UserProfilePropsInterface) {
  // ...same implementation
  return <div>{username}</div>;
}

// Using `type` (recommended for consistency)
type UserProfilePropsType = {
  username: string;
  avatarUrl: string;
  isActive: boolean;
};

function UserProfileWithType({ username, avatarUrl, isActive }: UserProfilePropsType) {
  // ...same implementation
  return <div>{username}</div>;
}
```

For typing component props, both are functionally equivalent. The recommendation to stick with `type` is for consistency across your codebase, as `type` can represent unions and other complex types that `interface` cannot.

### Typing Complex Prop Shapes

Props aren't just primitives. They can be objects, arrays, and functions.

```jsx
import React from 'react';

type User = {
  id: number;
  name: string;
};

type Post = {
  id: number;
  title: string;
  author: User; // Nested object
  tags: string[]; // Array of strings
  onLike: (postId: number) => void; // Function prop
};

function PostCard({ post }: { post: Post }) {
  const handleLike = () => {
    // We can call `onLike` with confidence, knowing it exists and
    // expects a number.
    post.onLike(post.id);
  };

  return (
    <div>
      <h3>{post.title}</h3>
      <p>By: {post.author.name}</p>
      <div>
        {post.tags.map(tag => <span key={tag}>#{tag} </span>)}
      </div>
      <button onClick={handleLike}>Like</button>
    </div>
  );
}
```

Here, we've defined a complex `Post` type that includes:

- A nested `User` object.
- An array of strings for `tags`.
- A function `onLike` that takes a `postId` and returns nothing (`void`).

By defining this shape, we get full type safety and autocompletion when accessing `post.author.name` or calling `post.onLike(post.id)`.

### Production Perspective

- **Colocate Types**: For components that are only used in one file, define the props type right above the component. For shared components, it's common to have a `types.ts` file in the component's folder or to export the type from the component file itself (more on this in 20.6).
- **Props as an API Contract**: Think of your props type as the formal documentation for your component. It tells other developers exactly what data is needed and in what shape.
- **Avoid `any`**: Never use `any` in your props. It's an escape hatch that negates the benefits of TypeScript. If you have a complex object you don't want to type fully, use `unknown` and perform type checking, or at least use a generic `Record<string, unknown>`.

## Props with Children

## Learning Objective

Correctly type components that accept `children` props using the explicit `React.ReactNode` type.

## Why This Matters

Many components are designed to wrap other content, like layouts, cards, or modals. As of React 19's type definitions, you must be explicit about accepting `children`. Mastering this pattern is essential for building compositional UIs in a type-safe way.

## Discovery Phase

Let's build a `Card` component. It should be able to wrap any content we pass to it.

```jsx
import React from 'react';

// Step 1: Define the props, including `children`.
// `React.ReactNode` is the correct type for anything React can render.
type CardProps = {
  title: string;
  children: React.ReactNode;
};

// Step 2: Use the props in your component.
function Card({ title, children }: CardProps) {
  const style = {
    border: '1px solid #ccc',
    borderRadius: '8px',
    padding: '16px',
    margin: '10px',
  };

  return (
    <div style={style}>
      <h2>{title}</h2>
      {children}
    </div>
  );
}

// Step 3: Use the component by nesting content inside it.
export default function App() {
  return (
    <Card title="User Profile">
      <p>This is some information about the user.</p>
      <button>Contact</button>
    </Card>
  );
}
```

**Rendered Output**:

```
(A bordered box with the title "User Profile" containing a paragraph and a button.)
```

The key here is `children: React.ReactNode;`. By adding this to our `CardProps`, we explicitly state that this component is designed to be a container. If we were to omit it, trying to pass children to `<Card>` would result in a TypeScript error.

## Deep Dive

### What is `React.ReactNode`?

`React.ReactNode` is a very broad type provided by React. It includes:

- `string`
- `number`
- `boolean` (which don't render)
- `null` and `undefined` (which don't render)
- `React.ReactElement` (i.e., JSX like `<p />`)
- An array of `React.ReactNode`s

This makes it the perfect type for `children` because it covers every valid value you could possibly place between a component's opening and closing tags.

### Optional Children

Sometimes, a component can accept children but doesn't require them. In this case, you can make the `children` prop optional with a `?`.

```jsx
import React from 'react';

type AlertBoxProps = {
  type: 'info' | 'warning';
  message: string;
  children?: React.ReactNode; // Children are optional
};

function AlertBox({ type, message, children }: AlertBoxProps) {
  const colors = {
    info: '#e0f7fa',
    warning: '#fff3e0',
  };

  return (
    <div style={{ backgroundColor: colors[type], padding: '10px' }}>
      <strong>{message}</strong>
      {children && <div style={{ marginTop: '8px' }}>{children}</div>}
    </div>
  );
}

export default function App() {
  return (
    <div>
      {/* Used without children */}
      <AlertBox type="warning" message="Your session is about to expire." />

      {/* Used with children */}
      <AlertBox type="info" message="New feature available!">
        <p>You can now export your data to CSV.</p>
      </AlertBox>
    </div>
  );
}
```

By marking `children` as optional (`children?`), we can use `AlertBox` in both ways. Inside the component, we check if `children` exists before rendering it to avoid rendering an empty container.

### Common Confusion: `React.ReactNode` vs. `React.ReactElement`

**You might think**: "My children will always be JSX, so shouldn't I use `React.ReactElement`?"

**Actually**: `React.ReactElement` is much more restrictive. It only accepts a single JSX element. It does _not_ accept strings, numbers, or arrays of elements.

```typescript
const element: React.ReactElement = <div />; // OK
const text: React.ReactElement = "Hello"; // Error!
const array: React.ReactElement = [<p />, <p />]; // Error!
```

**Why the confusion happens**: The names are similar, and in many cases, the child is just one element.

**How to remember**: Always default to `React.ReactNode` for `children`. It's safer and covers all valid use cases. Only use `React.ReactElement` if you are building a component that _must_ receive a single JSX element and nothing else (e.g., a component that clones its child to add props).

### Production Perspective

- **Composition is Key**: The `children` prop is the heart of React's composition model. Typing it correctly is fundamental to building reusable and flexible UI libraries.
- **Clarity of Intent**: Explicitly typing `children` makes the component's purpose clear. A developer seeing `children: React.ReactNode` in the props knows immediately that it's a wrapper or container component.
- **The `PropsWithChildren` Utility Type**: You might see `type MyProps = PropsWithChildren<{ foo: string }>` in older code. This was a helper type that automatically added `children` for you. It is now discouraged in favor of explicitly adding `children: React.ReactNode` to your own props type for better clarity.

## Ref Props Without forwardRef

## Learning Objective

Pass `ref`s to function components as regular props in React 19, eliminating the need for the `React.forwardRef` wrapper.

## Why This Matters

Directly passing `ref`s has been a long-standing request in the React community. React 19 finally makes it possible, simplifying component code, reducing boilerplate, and making `ref`s behave like any other prop. This is a major quality-of-life improvement.

## Discovery Phase

Let's look at the classic use case: creating a custom `Input` component that the parent needs to focus programmatically.

First, let's quickly review the **legacy way** using `forwardRef`.

```jsx
// LEGACY PATTERN: Pre-React 19
import React, { forwardRef } from 'react';

type LegacyInputProps = {
  label: string;
};

// We had to wrap our entire component in `forwardRef`.
const LegacyInput = forwardRef<HTMLInputElement, LegacyInputProps>(
  ({ label }, ref) => {
    return (
      <div>
        <label>{label}</label>
        <input ref={ref} />
      </div>
    );
  }
);
```

This pattern worked, but it was verbose. The `forwardRef` wrapper and the separate `ref` argument were confusing for beginners.

Now, let's see the beautiful simplicity of the **modern React 19 way**.

```jsx
import React, { useRef, useEffect } from 'react';

// MODERN PATTERN: React 19
// Step 1: Add `ref` to your props type.
// Use `React.Ref<T>` where T is the DOM element type.
type ModernInputProps = {
  label: string;
  ref: React.Ref<HTMLInputElement>;
};

// Step 2: Your component is just a regular function.
// `ref` is destructured from props like anything else.
function ModernInput({ label, ref }: ModernInputProps) {
  return (
    <div>
      <label>{label}</label>
      <input ref={ref} />
    </div>
  );
}

// --- USAGE ---
export default function App() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    // Focus the input when the component mounts
    inputRef.current?.focus();
  }, []);

  return (
    <div>
      <p>The input below will be focused on page load.</p>
      <ModernInput label="My Input" ref={inputRef} />
    </div>
  );
}
```

**Interactive Behavior**:
When the page loads, the input field will be automatically focused, and the cursor will be blinking inside it.

The difference is stark.

1.  No more `forwardRef` wrapper.
2.  `ref` is just another property in our `ModernInputProps` type.
3.  The component is a standard function, making it easier to read and type.

The correct type to use is `React.Ref<T>`, where `T` is the type of the element the ref will be attached to (e.g., `HTMLInputElement`, `HTMLButtonElement`, `HTMLDivElement`).

## Deep Dive

### How Does It Work?

React 19 changed the compiler and reconciler to treat a prop named `ref` specially. When you pass a `ref` prop to a function component, React understands that it should not be treated as a regular prop but should be attached to the underlying DOM element you specify. This was previously only possible with class components or `forwardRef`.

### Typing Refs for Custom Component Instances

You can also use `ref` to get a handle on a custom component instance, allowing you to call methods on it. This is done via the `useImperativeHandle` hook.

```jsx
import React, { useRef, useImperativeHandle } from 'react';

// Step 1: Define the methods you want to expose.
export type FancyInputHandle = {
  fancyFocus: () => void;
  fancyClear: () => void;
};

// Step 2: Type the ref prop with your handle type.
type FancyInputProps = {
  label: string;
  ref: React.Ref<FancyInputHandle>;
};

function FancyInput({ label, ref }: FancyInputProps) {
  const inputRef = useRef<HTMLInputElement>(null);

  // Step 3: Use `useImperativeHandle` to expose specific functions.
  useImperativeHandle(ref, () => ({
    fancyFocus() {
      inputRef.current?.focus();
      console.log('Focused with a fancy border!');
      inputRef.current?.style.setProperty('border', '2px solid red');
    },
    fancyClear() {
      if (inputRef.current) {
        inputRef.current.value = '';
        inputRef.current.style.setProperty('border', '1px solid black');
      }
    },
  }));

  return (
    <div>
      <label>{label}</label>
      <input ref={inputRef} />
    </div>
  );
}

// --- USAGE ---
export default function App() {
  const fancyRef = useRef<FancyInputHandle>(null);

  return (
    <div>
      <FancyInput label="Fancy Input" ref={fancyRef} />
      <button onClick={() => fancyRef.current?.fancyFocus()}>Focus It</button>
      <button onClick={() => fancyRef.current?.fancyClear()}>Clear It</button>
    </div>
  );
}
```

In this advanced pattern, we type the `ref` with `React.Ref<FancyInputHandle>`. This tells the parent component that when it uses `fancyRef.current`, it will have access to the `fancyFocus` and `fancyClear` methods, and nothing else. This is a powerful way to create safe and explicit APIs for component interaction.

### Production Perspective

- **Migration Path**: When upgrading a codebase to React 19, gradually refactor all `forwardRef` components to the new, simpler pattern. This is a low-risk, high-reward change that improves code clarity.
- **Naming Convention**: Always name the prop `ref`. React 19 specifically looks for a prop with this exact name to enable the new behavior. Don't use names like `inputRef` or `forwardedRef` for this purpose anymore.
- **Limit Imperative APIs**: While exposing methods via `useImperativeHandle` is powerful, it should be used sparingly. It's an "imperative" escape hatch in a declarative library. Always prefer declarative solutions (passing props) when possible. Use imperative handles for things that are difficult to express with props, like managing focus, triggering animations, or integrating with third-party DOM libraries.

## Optional and Default Props

## Learning Objective

Define optional props in a type definition and provide default values for them within the component using JavaScript's default parameters.

## Why This Matters

Components should be flexible. Optional props allow a component to have a simpler default appearance or behavior, which can be customized when needed. Providing sensible defaults makes your components easier to use and reduces the amount of boilerplate code required from the consumer.

## Discovery Phase

Let's build a `Button` component. Most of the time, it will be a standard button, but we want to allow for optional customization, like changing its size or disabling it.

```jsx
import React from 'react';

// Step 1: Define optional props with `?`
type ButtonProps = {
  children: React.ReactNode;
  onClick: () => void;
  size?: 'small' | 'medium' | 'large'; // Optional
  disabled?: boolean; // Optional
};

// Step 2: Provide default values using default parameters
function Button({
  children,
  onClick,
  size = 'medium', // Default value
  disabled = false, // Default value
}: ButtonProps) {
  const sizeStyles = {
    small: { padding: '4px 8px', fontSize: '12px' },
    medium: { padding: '8px 16px', fontSize: '14px' },
    large: { padding: '12px 24px', fontSize: '16px' },
  };

  return (
    <button
      onClick={onClick}
      disabled={disabled}
      style={sizeStyles[size]}
    >
      {children}
    </button>
  );
}

// --- USAGE ---
export default function App() {
  return (
    <div style={{ display: 'flex', alignItems: 'center', gap: '10px' }}>
      {/* Uses all defaults (medium size, not disabled) */}
      <Button onClick={() => alert('Default clicked!')}>Default</Button>

      {/* Overrides the size prop */}
      <Button onClick={() => alert('Large clicked!')} size="large">
        Large
      </Button>

      {/* Overrides the disabled prop */}
      <Button onClick={() => alert('Disabled clicked!')} disabled={true}>
        Disabled
      </Button>
    </div>
  );
}
```

**Rendered Output**:

```
[Default] [  Large  ] [Disabled]
(Three buttons of different sizes and states.)
```

This pattern is elegant and effective:

1.  In `ButtonProps`, `size?` and `disabled?` tell TypeScript that these props are optional. A consumer of `Button` does not have to provide them.
2.  In the function signature, `size = 'medium'` and `disabled = false` are JavaScript default parameters. If the `size` or `disabled` prop is `undefined` (i.e., not passed by the parent), these values will be used instead.

This combination ensures that inside our component, `size` and `disabled` are _always_ defined. We don't need to check for `undefined`.

## Deep Dive

### Why Default Parameters are a Best Practice

You could achieve a similar result by checking for `undefined` inside the component body:

```typescript
function Button(props: ButtonProps) {
  const { children, onClick } = props;
  const size = props.size ?? "medium";
  const disabled = props.disabled ?? false;
  // ...
}
```

However, using default parameters in the function signature is generally preferred because:

- **It's more concise**: The declaration and default value are in one place.
- **It's clearer to readers**: It's immediately obvious what the default values are just by looking at the function signature.
- **It plays nicely with destructuring**: It fits naturally into the destructuring pattern we already use for props.

### Common Confusion: `null` vs. `undefined` for Default Props

**You might think**: "If I pass `size={null}` to the button, will it use the default value?"

**Actually**: No. JavaScript default parameters only kick in if the value is `undefined`. If you explicitly pass `null`, the variable will be `null`.

**Why the confusion happens**: `null` and `undefined` are often treated similarly.

**How to remember**: Default parameters answer the question, "What should the value be if this prop is _missing_?" If the prop is explicitly provided (even as `null`), that value will be used. This is why it's important to have strict `tsconfig.json` settings, as TypeScript would likely catch an attempt to use `null` where a string like `'medium'` is expected.

### Production Perspective

- **Design for Simplicity**: When designing a component API, make the most common use case the one with the fewest props. Use optional props with sensible defaults for customizations. A good component should be easy to use for the simple case and powerful enough for the complex case.
- **Boolean Props**: For boolean props like `disabled`, `isLoading`, or `isActive`, it's a common convention to make them optional and default to `false`. This allows consumers to use the shorthand `<MyComponent disabled />` which is equivalent to `disabled={true}`.
- **Consistency in Defaults**: If you have a library of components, be consistent with your default props. For example, all buttons might default to `size="medium"`. This makes the library predictable and easier to learn.

## Component Type Exports

## Learning Objective

Export component prop types alongside the component itself to enable type-safe composition and usage in other parts of the application.

## Why This Matters

When you build a component, you're not just building the runtime code; you're also creating a typed API. Exporting the props type allows other developers (or you, in another file) to use your component without having to guess its props or re-declare its types. This is fundamental for building scalable and maintainable frontends.

## Discovery Phase

Let's create a `FancyButton` component and its corresponding props type in its own file. Then we'll see how to import and use both in another file.

```jsx
// FileName: components/FancyButton.tsx

import React from 'react';

// Step 1: Export the props type directly.
export type FancyButtonProps = {
  variant: 'primary' | 'secondary';
  children: React.ReactNode;
  onClick: () => void;
};

// Step 2: Export the component as the default export.
export default function FancyButton({ variant, children, onClick }: FancyButtonProps) {
  const style = {
    backgroundColor: variant === 'primary' ? 'blue' : 'grey',
    color: 'white',
    border: 'none',
    padding: '10px 20px',
  };

  return (
    <button style={style} onClick={onClick}>
      {children}
    </button>
  );
}
```

Now, let's imagine we are building a `CallToAction` component in a different file. This component will use our `FancyButton`, and it needs to accept some of the button's props to pass them through.

```jsx
// FileName: components/CallToAction.tsx

import React from 'react';
// Step 3: Import the component AND its props type.
import FancyButton, { type FancyButtonProps } from './FancyButton';

// We can use `Pick` to select only the props we want to pass through.
type CallToActionProps = {
  title: string;
  buttonVariant: FancyButtonProps['variant']; // Or Pick<FancyButtonProps, 'variant'>
};

export default function CallToAction({ title, buttonVariant }: CallToActionProps) {
  return (
    <div>
      <h1>{title}</h1>
      <FancyButton variant={buttonVariant} onClick={() => console.log('CTA Clicked!')}>
        Click Me
      </FancyButton>
    </div>
  );
}
```

This pattern is incredibly powerful.

1.  We exported `FancyButtonProps` from `FancyButton.tsx`.
2.  In `CallToAction.tsx`, we imported `FancyButtonProps`. The `type` keyword in the import statement (`import { type FancyButtonProps }`) is a modern feature that tells bundlers this is a type-only import, which can be safely erased at compile time.
3.  We then reused `FancyButtonProps['variant']` to ensure that our `CallToAction` component's `buttonVariant` prop must be one of the valid variants from the button itself. If we later add a `'tertiary'` variant to `FancyButton`, this type will update automatically.

## Deep Dive

### Patterns for Exporting Types

There are a few common conventions for exporting types:

1.  **Named export from component file (Recommended)**: This is the pattern we just saw. It's clean, simple, and keeps the types co-located with the component logic.

    ```typescript
    // Button.tsx
    export type ButtonProps = { /* ... */ };
    export default function Button(props: ButtonProps) { /* ... */ }
    ```
    
    2.  **Separate `types.ts` file**: For very complex components with many related types, you might create a dedicated types file.

    ```
    - Button/
      - index.tsx
      - types.ts
    ```
    
    ```typescript
    // Button/types.ts
    export type ButtonProps = { /* ... */ };
    export type ButtonSize = 'small' | 'medium';
    ```

    ```typescript
    // Button/index.tsx
    import { ButtonProps } from "./types";
    export * from "./types"; // Re-export types for consumers
    export default function Button(props: ButtonProps) {
      /* ... */
    }
    ```

    This approach is good for organization when you have more than just one props type.

### Using `React.ComponentProps`

TypeScript gives us a handy utility type for when a component _doesn't_ export its props type. `React.ComponentProps<T>` can extract the props type from a component type.

```javascript
import React from 'react';

// Imagine this component is from a third-party library and doesn't export its props.
function ThirdPartyButton({ size }: { size: 'sm' | 'lg' }) {
  return <button>{size}</button>;
}

// We can extract its props type!
type InferredProps = React.ComponentProps<typeof ThirdPartyButton>;
// InferredProps is now: { size: "sm" | "lg"; }

function MyWrapper() {
  const props: InferredProps = { size: 'lg' }; // This is type-safe
  return <ThirdPartyButton {...props} />;
}
```

While this is a useful escape hatch, it's always better to explicitly export props types from your own components. `React.ComponentProps` is most useful for working with third-party libraries that haven't followed this best practice.

### Production Perspective

- **Building a Design System**: When creating a reusable component library or design system, exporting prop types is non-negotiable. It's the primary way you provide a type-safe API for the consumers of your library.
- **Wrapper Components**: The most common use case for imported prop types is creating wrapper components. A wrapper might add some styling, state, or logic around a base component and needs to accept and pass through a subset of the base component's props. Using `Pick` and `Omit` with the imported prop types is the standard way to do this.
- **Type-Only Imports**: Use `import type { MyType } from './MyFile'` or `import { type MyType } from './MyFile'`. This makes it explicit that you're importing something that has no runtime code, which can help with build optimizations and prevents accidental imports of values when you only wanted a type.

## Event Handler Types

## Learning Objective

Correctly type event handler props and inline event callbacks using React's synthetic event types for full type safety on event objects.

## Why This Matters

Handling user interaction is at the core of React. Typing your event handlers correctly ensures you can safely access event properties like `event.target.value` without runtime errors and gives you excellent autocompletion in your editor, speeding up development.

## Discovery Phase

Let's start with the most common event: a button click.

```jsx
import React from 'react';

// We want to pass a function to be called on click.
type ClickableButtonProps = {
  onButtonClick: (message: string) => void;
};

function ClickableButton({ onButtonClick }: ClickableButtonProps) {
  // To get the type for the `event` object, we can use React's built-in types.
  // The type for a click event is `React.MouseEvent`.
  // We can make it more specific by providing the element type in angle brackets.
  const handleClick = (event: React.MouseEvent<HTMLButtonElement>) => {
    // Now, `event` is fully typed! We get autocomplete for properties like:
    // event.currentTarget
    // event.clientX
    // event.altKey
    console.log('Button was clicked!', event.currentTarget);
    onButtonClick('Button clicked successfully!');
  };

  return <button onClick={handleClick}>Click Me</button>;
}
```

The key is `React.MouseEvent<HTMLButtonElement>`.

- `MouseEvent` tells us it's a mouse-related event (click, mousedown, mouseup, etc.).
- `<HTMLButtonElement>` is a generic argument that makes the type even more specific. It tells TypeScript that `event.currentTarget` will be an HTML button element, giving us access to all of its properties.

Now let's look at a form input's `onChange` event, which is slightly different.

```jsx
import React, { useState } from 'react';

function ControlledInput() {
  const [value, setValue] = useState('');

  // The type for an input's change event is `React.ChangeEvent`.
  // The specific element is `HTMLInputElement`.
  const handleChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    // Because we typed the event correctly, TypeScript knows that
    // `event.target` is an input element and has a `value` property.
    setValue(event.target.value);
  };

  return (
    <div>
      <input type="text" value={value} onChange={handleChange} />
      <p>Current value: {value}</p>
    </div>
  );
}
```

**Interactive Behavior**:
As you type in the input field, the text below it updates in real-time.

Without `React.ChangeEvent<HTMLInputElement>`, TypeScript would not know about `event.target.value`, and you would get a compile-time error. This is one of the most common places where proper typing prevents runtime errors.

## Deep Dive

### Common Event Types

Here is a quick reference for the most common event types you'll use:

| Event                    | Type                  | Element Generic                                                |
| :----------------------- | :-------------------- | :------------------------------------------------------------- |
| `onClick`, `onMouseDown` | `React.MouseEvent`    | `HTMLButtonElement`, `HTMLDivElement`, etc.                    |
| `onChange`               | `React.ChangeEvent`   | `HTMLInputElement`, `HTMLTextAreaElement`, `HTMLSelectElement` |
| `onSubmit`               | `React.FormEvent`     | `HTMLFormElement`                                              |
| `onFocus`, `onBlur`      | `React.FocusEvent`    | Any focusable element                                          |
| `onKeyDown`, `onKeyUp`   | `React.KeyboardEvent` | Any element that can receive key events                        |

**Pro Tip**: You don't need to memorize these. In VS Code, you can simply hover your mouse over the event prop (e.g., `onClick` in the JSX) and it will show you the exact type signature required.

### Typing Event Handler Props

When a parent component needs to provide the event handler, you type it in the component's props.

```jsx
import React from 'react';

type SearchBarProps = {
  // The prop is a function that receives the new text value.
  onSearchChange: (newQuery: string) => void;
};

function SearchBar({ onSearchChange }: SearchBarProps) {
  const handleChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    onSearchChange(event.target.value);
  };

  return (
    <input
      type="search"
      placeholder="Search..."
      onChange={handleChange}
    />
  );
}
```

Notice the separation of concerns. The `SearchBar` component knows how to handle the raw browser event (`React.ChangeEvent`). It then extracts the useful data (`event.target.value`) and calls the `onSearchChange` prop with just that data. This is a great pattern because the parent component doesn't need to know about the browser's event object; it just receives the string it cares about.

### Production Perspective

- **Abstracting Event Details**: As shown in the `SearchBar` example, it's a best practice for child components to handle the raw event and pass up only the relevant data to the parent. This makes parent components simpler and decouples them from the child's internal DOM structure.
- **Passing the Full Event**: Sometimes, the parent component _does_ need the full event object (e.g., to call `event.preventDefault()`). In that case, you would type the prop like this:
  ```typescript
  type FormProps = {
    // The prop is a function that receives the entire event object.
    onSubmit: (event: React.FormEvent<HTMLFormElement>) => void;
  };
  ```
- **Inline Handlers**: For simple cases, you can write the handler inline. TypeScript's inference is so good that you often don't need to type the `event` parameter manually.
  ```jsx
  <input onChange={(e) => setValue(e.target.value)} />
  ```
  Here, `e` is automatically inferred as `React.ChangeEvent<HTMLInputElement>` because TypeScript knows it's being passed to the `onChange` prop of an `<input>` element.

## Typing Refs

## Learning Objective

Correctly type refs created with the `useRef` hook for accessing DOM elements and persisting mutable values across renders.

## Why This Matters

Refs are the primary way to interact with the DOM directly in React. Proper typing is crucial for ensuring that you are calling methods on the correct element type (e.g., `.focus()` on an input) and for handling the fact that the ref's `current` property is `null` on the initial render.

## Discovery Phase

Let's revisit the focus management example, but this time, the ref will be created and used entirely _inside_ one component.

```jsx
import React, { useRef, useEffect } from "react";

function FocusableInput() {
  // Step 1: Provide a generic argument to `useRef` for the element type.
  // Initialize it with `null`.
  const inputRef = useRef < HTMLInputElement > null;

  useEffect(() => {
    // Step 2: The `current` property might be null.
    // On the first render, React hasn't attached the ref to the DOM element yet.
    // The effect runs *after* the render, so `current` will be populated.
    // We use optional chaining (`?.`) for safety.
    inputRef.current?.focus();
  }, []); // Empty dependency array means this runs once after mount.

  return (
    <div>
      <p>This input will be focused after the component mounts.</p>
      {/* Step 3: Attach the ref to the DOM element. */}
      <input ref={inputRef} type="text" />
    </div>
  );
}
```

This example highlights the three critical steps for typing DOM refs:

1.  **Declaration**: `useRef<HTMLInputElement>(null)`. You tell `useRef` what kind of element it will hold. The initial value _must_ be `null` for DOM refs, because the DOM node doesn't exist yet when `useRef` is called.
2.  **Access**: `inputRef.current?.focus()`. The type of `inputRef.current` is `HTMLInputElement | null`. TypeScript forces you to account for the `null` case. Optional chaining (`?.`) is the most common way to handle this.
3.  **Attachment**: `<input ref={inputRef} />`. You pass the ref object to the `ref` attribute of the DOM element.

## Deep Dive

### The Ref's `current` Property

The `ref` object itself is stable across renders. It's a simple object with a single property: `current`.

```javascript
// The shape of a ref object
{
  current: T | null // where T is the type you provided
}
```

React mutates the `current` property to point to the DOM node after it has been mounted. Before that, it's `null`. This is why the null check is essential.

### Refs for Storing Mutable Values

Refs can also be used to store any mutable value that you want to persist across renders without causing a re-render when it changes. This is like an "instance variable" for a function component.

When using `useRef` for this purpose, you can provide a non-null initial value.

```jsx
import React, { useRef, useEffect, useState } from 'react';

function Timer() {
  const [seconds, setSeconds] = useState(0);

  // Here, we're storing a number. The type is inferred.
  // `intervalRef.current` will be `number | undefined`.
  const intervalRef = useRef<number>();

  useEffect(() => {
    // Start the interval
    intervalRef.current = window.setInterval(() => {
      setSeconds(prev => prev + 1);
    }, 1000);

    // Cleanup function: clear the interval when the component unmounts.
    return () => {
      window.clearInterval(intervalRef.current);
    };
  }, []); // Run only once on mount.

  return <h1>Timer: {seconds}s</h1>;
}
```

In this example, `intervalRef` is used to hold the ID of the `setInterval`. We need to store this ID so we can clear it in the `useEffect` cleanup function when the component unmounts.

- We type it as `useRef<number>()`. Since we don't provide an initial value, it starts as `undefined`.
- The type of `intervalRef.current` is `number | undefined`.
- This is a perfect use case for a ref because changing the interval ID should not trigger a re-render.

### Common Confusion: State vs. Ref for Mutable Data

**You might think**: "If I need to store a value, why not just use `useState`?"

**Actually**: You should use `useState` only for data that, when changed, should cause the component to re-render and reflect the new value in the UI. Use `useRef` for data that needs to be persisted across renders but does _not_ directly drive the UI.

**Why the confusion happens**: Both can "hold" data.

**How to remember**:

- Does changing this value need to update what the user sees? → Use **state**.
- Do I just need to keep track of something behind the scenes (like a timer ID, a previous value, or a third-party library instance)? → Use **ref**.

### Production Perspective

- **DOM Manipulation**: Use refs for direct DOM interactions that are hard to achieve declaratively, such as managing focus, measuring element dimensions, or integrating with non-React libraries that need a DOM node (like a charting library).
- **Avoiding Null Checks**: In some cases, you might be certain that a ref will be populated by the time you access it (e.g., in an event handler that only runs after a user clicks). In these rare cases, you can use a non-null assertion (`!`) like `myRef.current!.focus()`. However, this is risky as it bypasses TypeScript's safety checks. Use optional chaining (`?.`) whenever possible.
- **Refs and Custom Hooks**: Refs are often encapsulated within custom hooks to manage complex logic, such as a `useClickOutside` hook that takes a ref to an element and triggers a callback when the user clicks outside of it.

## Generic Components

## Learning Objective

Create flexible, reusable, and type-safe generic components that can work with various data structures while maintaining full type safety.

## Why This Matters

Hard-coding components to specific data types leads to code duplication and poor maintainability. Generics are the key to unlocking true reusability in a typed system, allowing you to build foundational components (like lists, tables, and dropdowns) that can be adapted to any data model in your application.

## Discovery Phase

In Chapter 19, we built a basic generic `List` component. Let's take it a step further and build a generic `SelectDropdown` component. This component should be able to handle an array of any kind of object, as long as those objects have some properties we can use for the value and label of the options.

```jsx
import React, { useState } from 'react';

// Step 1: Define a generic constraint.
// Any object passed to our component MUST have `id` and `name` properties.
type SelectableItem = {
  id: string | number;
  name: string;
};

// Step 2: Create the generic props type.
// `T` can be any object, as long as it `extends` our constraint.
type SelectDropdownProps<T extends SelectableItem> = {
  items: T[];
  label: string;
  onSelect: (selectedItem: T) => void;
};

// Step 3: Create the generic component.
export function SelectDropdown<T extends SelectableItem>({
  items,
  label,
  onSelect,
}: SelectDropdownProps<T>) {
  const [selectedValue, setSelectedValue] = useState<string>('');

  const handleChange = (event: React.ChangeEvent<HTMLSelectElement>) => {
    const selectedId = event.target.value;
    const selectedItem = items.find(item => String(item.id) === selectedId);
    setSelectedValue(selectedId);
    if (selectedItem) {
      onSelect(selectedItem);
    }
  };

  return (
    <div>
      <label>{label}</label>
      <select value={selectedValue} onChange={handleChange}>
        <option value="" disabled>Select an option</option>
        {items.map(item => (
          // We can safely access `item.id` and `item.name` because of our constraint.
          <option key={item.id} value={item.id}>
            {item.name}
          </option>
        ))}
      </select>
    </div>
  );
}
```

Now, let's use this generic component with two different data structures.

```jsx
// --- USAGE ---

// Data structure 1: Users
const users = [
  { id: 1, name: 'Alice', role: 'admin' },
  { id: 2, name: 'Bob', role: 'user' },
];

// Data structure 2: Products
const products = [
  { id: 'p1', name: 'Laptop', price: 1500 },
  { id: 'p2', name: 'Keyboard', price: 100 },
];

export default function App() {
  const handleUserSelect = (user: { id: number; name: string; role: string }) => {
    // `user` is fully typed here! We can access `user.role`.
    alert(`Selected user: ${user.name}, Role: ${user.role}`);
  };

  const handleProductSelect = (product: { id: string; name: string; price: number }) => {
    // `product` is also fully typed! We can access `product.price`.
    alert(`Selected product: ${product.name}, Price: $${product.price}`);
  };

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '20px' }}>
      <SelectDropdown items={users} label="Select User" onSelect={handleUserSelect} />
      <SelectDropdown items={products} label="Select Product" onSelect={handleProductSelect} />
    </div>
  );
}
```

**Interactive Behavior**:
The UI shows two dropdowns. When you select an option from the "Select User" dropdown, an alert shows the user's name and role. When you select from the "Select Product" dropdown, an alert shows the product's name and price.

This demonstrates the power of generics:

1.  **Constraint**: `T extends { id: string | number; name: string; }` ensures that any data we pass in is workable.
2.  **Flexibility**: We can pass an array of `users` or `products`, and the component works perfectly with both.
3.  **Type Safety**: In the `onSelect` callback, TypeScript knows the exact type of the selected item (`user` or `product`), including the properties that were not part of the constraint (like `role` and `price`).

## Deep Dive

### Making Generic Components Even More Flexible

Our `SelectDropdown` requires the properties to be named `id` and `name`. What if our data uses `value` and `label` instead? We can make the component even more generic by allowing the consumer to provide accessor functions.

```jsx
import React from 'react';

// No constraint on T this time!
type SuperSelectProps<T> = {
  items: T[];
  // Functions to get the value and label from an item of type T
  getItemValue: (item: T) => string | number;
  getItemLabel: (item: T) => string;
  onSelect: (selectedItem: T) => void;
};

export function SuperSelect<T>({
  items,
  getItemValue,
  getItemLabel,
  onSelect,
}: SuperSelectProps<T>) {
  // ... implementation would be similar, but use the accessor functions
  return (
    <select onChange={(e) => {
      const value = e.target.value;
      const selected = items.find(item => String(getItemValue(item)) === value);
      if (selected) onSelect(selected);
    }}>
      {items.map((item, index) => {
        const value = getItemValue(item);
        const label = getItemLabel(item);
        return <option key={index} value={value}>{label}</option>
      })}
    </select>
  );
}

// --- USAGE with different data shape ---
const categories = [
  { categoryId: 'cat1', displayName: 'Electronics' },
  { categoryId: 'cat2', displayName: 'Books' },
];

// <SuperSelect
//   items={categories}
//   getItemValue={(c) => c.categoryId}
//   getItemLabel={(c) => c.displayName}
//   onSelect={(c) => alert(c.displayName)}
// />
```

This version is maximally flexible. It makes no assumptions about the shape of the data objects `T`. Instead, it asks the consumer to provide the logic for extracting the necessary pieces of data. This is a common pattern in advanced component libraries and rendering engines.

### Production Perspective

- **Identify the Generic Core**: When building a component, ask yourself: "What is the core, repeatable logic, and what is the data-specific part?" The core logic belongs in the generic component, and the data-specific parts should be handled by props (like `renderItem`) or accessor functions (`getItemValue`).
- **Start Specific, then Generalize**: It's often easier to build a specific version of a component first (e.g., a `UserSelectDropdown`). Once it's working, you can refactor it to be generic by identifying the hard-coded parts and replacing them with generic type parameters and props.
- **Generics in Custom Hooks**: Generics are not just for components. They are incredibly powerful for custom hooks. For example, a `useQuery<T>` hook could be generic over the type of data `T` it fetches from an API, providing end-to-end type safety from your data fetching layer to your components.

## Module Synthesis 📋

## Module Synthesis

In this chapter, we bridged the gap between TypeScript theory and the practical reality of building React components. We've established a comprehensive toolkit for typing every aspect of a component's API, ensuring they are robust, predictable, and easy for other developers to consume.

We began by defining the modern standard for typing function components, moving away from the legacy `React.FC` to a simpler, more explicit pattern. We then dove deep into the heart of component typing: defining **props**. We learned to type everything from simple primitives and union types to complex nested objects and callback functions.

We mastered the patterns for building compositional UIs by explicitly typing the **`children` prop** with `React.ReactNode`. We then celebrated a major React 19 improvement: passing **`ref`s as regular props**, which drastically simplifies creating components that need to expose DOM nodes. We also covered the essential patterns of **optional and default props**, making our components more flexible and easier to use.

Finally, we elevated our thinking to component architecture. We learned why **exporting prop types** is crucial for creating a scalable system of interconnected components. We covered how to type **event handlers** for safe user interactions and how to correctly type **`useRef`** for both DOM access and mutable value storage. We culminated with **generic components**, the pinnacle of reusability, allowing us to write a single component that can adapt to any data structure while maintaining perfect type safety.

## Looking Ahead

You are now fully equipped to write type-safe React components for almost any scenario. The next chapter, **Chapter 21: Typing React 19 Features**, will build directly on this knowledge. We will focus specifically on the new, state-of-the-art APIs introduced in React 19. We'll see how to type the revolutionary `use` hook, build type-safe forms with Actions using `useActionState` and `useOptimistic`, and understand the typing distinctions between Server and Client Components. This is where we apply our skills to the cutting edge of React.
