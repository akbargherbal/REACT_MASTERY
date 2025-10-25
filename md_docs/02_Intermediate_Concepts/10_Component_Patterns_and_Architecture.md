# Chapter 10: Component Patterns and Architecture

## Container and Presentational Components

## Learning Objective

Re-evaluate the classic "Container and Presentational" pattern in the context of React Server Components and hooks, and understand how modern React simplifies this separation of concerns.

## Why This Matters

The Container/Presentational pattern was one of the first major architectural patterns in React, designed to separate data fetching and logic (containers) from rendering and UI (presentational). While the original pattern is less common now, the core principle of separating concerns is more important than ever. Understanding how RSCs and hooks achieve this separation naturally will help you write cleaner, more maintainable code.

## Discovery Phase: The Classic Pattern

Let's look at a classic implementation of this pattern. We'd have a "smart" container component that knows how to fetch data and a "dumb" presentational component that just knows how to render it.

```jsx
// The "Dumb" Presentational Component
// It only receives props and renders UI. No data fetching, no state.
function UserProfileView({ user }) {
  if (!user) return <p>Loading...</p>;
  return (
    <div>
      <h1>{user.name}</h1>
      <p>Email: {user.email}</p>
    </div>
  );
}

// The "Smart" Container Component
// It handles logic, state, and data fetching.
class UserProfileContainer extends React.Component {
  state = { user: null };

  componentDidMount() {
    fetch(`/api/users/${this.props.userId}`)
      .then((res) => res.json())
      .then((user) => this.setState({ user }));
  }

  render() {
    return <UserProfileView user={this.state.user} />;
  }
}

// Usage: <UserProfileContainer userId="1" />
```

This pattern was revolutionary because it made `UserProfileView` highly reusable and easy to test. You could render it in a tool like Storybook just by passing it different `user` props. However, it created a lot of boilerplate, often requiring two files for one piece of UI.

## Deep Dive: The Modern Approach with RSCs and Hooks

Modern React provides more direct ways to achieve the same separation of concerns.

### Case 1: The Component is Purely for Data Display (No Interactivity)

If the component's job is just to fetch and display data, a **React Server Component** is the perfect modern equivalent. It naturally combines the data fetching and rendering logic in one place, but keeps that logic on the server.

```jsx
// This is a Server Component by default.
// It acts as its own data container.
async function UserProfile({ userId }) {
  // 1. Data fetching logic is co-located with the view.
  const user = await db.users.find(userId);

  // 2. Rendering logic uses the fetched data.
  return (
    <div>
      <h1>{user.name}</h1>
      <p>Email: {user.email}</p>
    </div>
  );
}

// Usage: <UserProfile userId="1" />
```

This single async component achieves the same goal as the original two-component pattern, but with far less code. The concern of "data fetching" is separated from the client by virtue of running on the server. The client receives only the final HTML, making this the ultimate "presentational" output.

### Case 2: The Component Has Interactive UI

What if the profile view also had a client-side button, like a "Follow" button? Here, we combine an RSC with a Client Component. The RSC acts as the container, and the Client Component is the interactive presentational piece.

```jsx
// The Server Component acts as the "Container"
// It fetches the data.
async function UserProfile({ userId }) {
  const user = await db.users.find(userId);

  return (
    // It passes the static data down to the client view.
    <UserProfileView user={user} />
  );
}

// The Client Component is the "Presentational" part
// It handles its own interactive state.
("use client");
import { useState } from "react";

function UserProfileView({ user }) {
  const [isFollowing, setIsFollowing] = useState(false);

  return (
    <div>
      <h1>{user.name}</h1>
      <p>Email: {user.email}</p>
      <button onClick={() => setIsFollowing(!isFollowing)}>
        {isFollowing ? "Following" : "Follow"}
      </button>
    </div>
  );
}
```

This is the modern evolution of the pattern.

- The **Server Component** is the data container.
- The **Client Component** is the interactive view.

The separation of concerns is now between the server and the client, which is a much clearer and more impactful boundary than the original class-based pattern.

## Production Perspective

- **RSCs as Natural Containers**: Think of your page-level Server Components as the primary data containers for your application. They fetch and distribute data to the rest of the tree.
- **Separating Logic with Hooks**: For complex client-side logic, the modern approach is not to create a separate container component, but to extract the logic into a custom hook (`useUserProfile`). The presentational component then uses this hook. This keeps the logic reusable without requiring two components.
- **When is the old pattern still useful?**: You will encounter the Container/Presentational pattern in older codebases. Understanding it is important for maintenance. However, for new development in React 19, the combination of Server Components for data and Client Components (with custom hooks for complex logic) is the superior approach.

## Higher-Order Components (HOCs)

## Learning Objective

Understand the Higher-Order Component (HOC) pattern, its original purpose, and why custom hooks have largely replaced it for logic sharing in modern React.

### Legacy Pattern Notice

**Pre-Hooks**: HOCs were a primary method for reusing component logic. They are a function that takes a component and returns a new, enhanced component.

**Modern React**: Custom Hooks provide a more direct and less complex way to share stateful logic. While HOCs are still used in some libraries and legacy code, they are no longer a recommended pattern for new application code.

## Discovery Phase: The HOC Pattern Explained

A Higher-Order Component is a function that wraps a component to provide it with extra props or behavior. A classic example is a `withUserData` HOC that fetches user data and injects it as a prop into the wrapped component.

```jsx
import React, { useState, useEffect } from "react";

// This is the HOC. It's a function that takes a component...
function withUserData(WrappedComponent, userId) {
  // ...and returns a new component.
  return function WithUserData(props) {
    const [user, setUser] = useState(null);

    useEffect(() => {
      fetch(`/api/users/${userId}`)
        .then((res) => res.json())
        .then((data) => setUser(data));
    }, [userId]);

    if (!user) return <p>Loading user data...</p>;

    // It renders the original component with the new `user` prop.
    return <WrappedComponent {...props} user={user} />;
  };
}

// A simple presentational component that knows nothing about fetching.
function UserGreeter({ user }) {
  return <h1>Hello, {user.name}!</h1>;
}

// We create the enhanced component by calling the HOC.
const UserGreeterWithData = withUserData(UserGreeter, "1");

// Usage: <UserGreeterWithData />
```

This pattern allowed us to reuse the data-fetching logic for any component that needed user data. However, it had several drawbacks:

- **Wrapper Hell**: Nesting multiple HOCs (`withTheme(withAuth(withUserData(MyComponent)))`) created deeply nested component trees that were hard to debug.
- **Prop Name Collisions**: If two HOCs both injected a prop named `data`, one would overwrite the other.
- **Implicit Dependencies**: It wasn't clear from looking at `UserGreeter` where the `user` prop was coming from.

## Deep Dive: The Modern Solution - Custom Hooks

Custom hooks solve the exact same problem—reusing stateful logic—in a much cleaner way. Let's rewrite our example using a `useUserData` custom hook.

```jsx
import React, { useState, useEffect } from "react";

// The custom hook encapsulates the logic.
function useUserData(userId) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then((data) => setUser(data));
  }, [userId]);

  return user;
}

// The component now explicitly uses the hook.
function UserGreeter({ userId }) {
  const user = useUserData(userId);

  if (!user) return <p>Loading user data...</p>;

  return <h1>Hello, {user.name}!</h1>;
}

// Usage: <UserGreeter userId="1" />
```

Comparing the two approaches reveals why hooks are superior:

- **No Wrapper Hell**: Hooks are just function calls inside your component. There's no extra nesting in the component tree.
- **No Prop Collisions**: The component gets the value back from the hook and can assign it to any variable name it wants (`const userData = useUserData(...)`).
- **Explicit Dependencies**: It's perfectly clear inside `UserGreeter` where the `user` data comes from. The dependency is explicit: `const user = useUserData(userId)`.
- **Better Composition**: You can use multiple hooks in a single component with no issues, and even use the output of one hook as the input to another.

## Production Perspective

- **When are HOCs still used?**: You will still encounter HOCs in the wild. Some libraries, especially older ones or those that need to manipulate component rendering in complex ways (like styling libraries or form libraries), still use them. For example, `React.memo` is technically an HOC.
- **The Migration Path**: When working in a legacy codebase, a common and valuable refactoring task is to convert HOCs into custom hooks. This often simplifies the code, improves type safety with TypeScript, and makes the component's logic easier to follow.
- **The RSC Alternative**: It's also worth noting that for the specific `withUserData` example, the absolute best modern solution is often a Server Component that simply `await`s the data, as we saw in the previous section. Custom hooks are the answer for sharing logic on the **client**.

## Render Props Pattern

## Learning Objective

Understand the Render Props pattern, its use case for sharing logic via props, and how custom hooks and component composition offer more direct alternatives.

### Legacy Pattern Notice

**Pre-Hooks**: The Render Props pattern was another popular technique for sharing stateful logic. It involves a component whose primary prop is a function that returns JSX.

**Modern React**: Like HOCs, this pattern has been largely superseded by custom hooks for logic sharing. However, the underlying idea of passing components as props remains a cornerstone of React composition.

## Discovery Phase: The Render Props Pattern Explained

A component using the Render Props pattern takes a function as a prop (commonly named `render` or passed as `children`). This function receives the state from the component and is responsible for rendering the UI.

Here is a `MouseTracker` component that uses a render prop to share the current mouse coordinates.

```jsx
"use client";
import React, { useState, useEffect } from "react";

// This component tracks the mouse position...
class MouseTracker extends React.Component {
  state = { x: 0, y: 0 };

  handleMouseMove = (event) => {
    this.setState({
      x: event.clientX,
      y: event.clientY,
    });
  };

  render() {
    return (
      <div
        style={{ height: "300px", border: "1px solid black" }}
        onMouseMove={this.handleMouseMove}
      >
        {/* ...and calls the render prop with its state. */}
        {this.props.render(this.state)}
      </div>
    );
  }
}

// The parent component decides what to render with the mouse coordinates.
function App() {
  return (
    <div>
      <h1>Move the mouse around!</h1>
      <MouseTracker
        render={({ x, y }) => (
          // This function is the "render prop".
          <p>
            The mouse position is ({x}, {y})
          </p>
        )}
      />
    </div>
  );
}
```

**Interactive Behavior**:
As you move your mouse over the bordered box, the text updates in real-time with the current X and Y coordinates.

The `MouseTracker` component encapsulated the logic of tracking the mouse, and the `render` prop allowed any parent to reuse that logic to render whatever UI it wanted. The main drawback was the added nesting and syntax complexity within the JSX.

## Deep Dive: Modern Alternatives

### 1. The Custom Hook Solution (for Logic Sharing)

Just like with HOCs, a custom hook is the modern, direct replacement for sharing this kind of logic.

```jsx
"use client";
import React, { useState, useEffect } from "react";

// The custom hook encapsulates the logic.
function useMousePosition() {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handleMouseMove = (event) => {
      setPosition({ x: event.clientX, y: event.clientY });
    };

    window.addEventListener("mousemove", handleMouseMove);
    return () => window.removeEventListener("mousemove", handleMouseMove);
  }, []);

  return position;
}

// The component that uses the logic is now much flatter.
function App() {
  const { x, y } = useMousePosition();

  return (
    <div>
      <h1>Move the mouse around!</h1>
      <p>
        The mouse position is ({x}, {y})
      </p>
    </div>
  );
}
```

This is clearly simpler. There's no extra component, no function passed as a prop, just a direct call to the hook that provides the needed values.

### 2. The Component Composition Solution (for "Slots")

The Render Props pattern wasn't just about logic; it was also about giving a component a "slot" to be filled by the parent. The modern way to do this is simply by passing JSX as `children`. This is the most fundamental composition pattern in React.

The `Collapsible` component we saw in Chapter 9 is a perfect example. It uses its `children` prop as a "render prop" for its content.

```jsx
// Client Component "Shell"
'use client';
import { useState } from 'react';

export default function Card({ children, title }) {
  // This component provides a "slot" for content via `children`.
  return (
    <div style={{ border: '1px solid gray', padding: '1em' }}>
      <h2>{title}</h2>
      {children}
    </div>
  );
}

// Server Component Parent
import Card from './Card';
import ServerInfo from './ServerInfo'; // An RSC

export default function Page() {
  return (
    <Card title="Server-Rendered Content">
      {/* We are filling the slot with another component. */}
      {/* This is a form of render prop, but much more natural. */}
      <ServerInfo />
    </Card>
  );
}
```

This is the true legacy of the Render Props pattern. While custom hooks replaced it for logic sharing, the core idea of passing renderable content as props (`children`) is more important than ever, especially for composing Server and Client components.

## Production Perspective

- **Logic Sharing**: For sharing stateful, non-visual logic, always prefer custom hooks.
- **UI Slots**: For creating reusable UI shells that need to be filled with content, use the `children` prop. This is the essence of component composition.
- **Legacy Code**: You will see the render prop pattern extensively in older codebases and some libraries (e.g., older versions of React Router or Formik). Understanding it is key to working with that code. The migration path is usually to refactor the logic into a custom hook and simplify the component that uses it.

## Compound Components

## Learning Objective

Implement the Compound Components pattern to create a set of components that work together to manage a shared, implicit state, providing a clean and expressive API for consumers.

## Why This Matters

Some UI elements have a complex relationship between a parent and its children. Think of a `<select>` and `<option>` pair, or a tabbed interface. The Compound Components pattern allows you to create a similar declarative API for your own components. It makes the components highly reusable and expressive, hiding the complex internal state management from the user.

## Discovery Phase: Building a Tab Interface

Let's build a set of `Tabs` components. The desired API should look like this:

```jsx
<Tabs>
  <TabList>
    <Tab>Tab 1</Tab>
    <Tab>Tab 2</Tab>
  </TabList>
  <TabPanels>
    <TabPanel>Content for tab 1</TabPanel>
    <TabPanel>Content for tab 2</TabPanel>
  </TabPanels>
</Tabs>
```

This API is clean and semantic. To make it work, all these components (`Tabs`, `Tab`, `TabPanel`, etc.) need to communicate with each other. A `Tab` needs to know which tab is currently active, and it needs to be able to set the active tab when clicked. This shared state is managed implicitly using React Context.

## Deep Dive: Implementing Compound Components with Context

Since this component is interactive, the state management must happen on the client. We'll create a `Tabs` component with the `'use client'` directive that acts as the context provider.

```jsx
"use client";

import React, { useState, useContext, createContext } from "react";

// 1. Create a context to hold the shared state.
const TabsContext = createContext(null);

// 2. The main parent component is the Provider.
export function Tabs({ children }) {
  const [activeIndex, setActiveIndex] = useState(0);

  return (
    <TabsContext.Provider value={{ activeIndex, setActiveIndex }}>
      {children}
    </TabsContext.Provider>
  );
}

// 3. Child components use the context to read and update state.
export function TabList({ children }) {
  // We can use React.Children to inject props like the index.
  const kids = React.Children.map(children, (child, index) => {
    return React.cloneElement(child, { index });
  });
  return (
    <div style={{ display: "flex", borderBottom: "1px solid black" }}>
      {kids}
    </div>
  );
}

export function Tab({ children, index }) {
  const { activeIndex, setActiveIndex } = useContext(TabsContext);
  const isActive = index === activeIndex;

  return (
    <button
      style={{
        padding: "1em",
        border: "none",
        background: isActive ? "lightblue" : "white",
        fontWeight: isActive ? "bold" : "normal",
      }}
      onClick={() => setActiveIndex(index)}
    >
      {children}
    </button>
  );
}

export function TabPanels({ children }) {
  const { activeIndex } = useContext(TabsContext);
  // Only render the active panel
  return <div>{React.Children.toArray(children)[activeIndex]}</div>;
}

export function TabPanel({ children }) {
  return <div>{children}</div>;
}
```

Now let's create a component that uses our new `Tabs` system. This parent component can be a Server Component because the interactivity is fully encapsulated within our client-side `Tabs` components.

```jsx
// We can import all the pieces from our client component file.
import { Tabs, TabList, Tab, TabPanels, TabPanel } from "./Tabs";

// This component can be a Server Component!
export default function ProductInfo() {
  return (
    <div>
      <h2>Product Information</h2>
      <Tabs>
        <TabList>
          <Tab>Description</Tab>
          <Tab>Specifications</Tab>
          <Tab>Reviews</Tab>
        </TabList>
        <TabPanels>
          <TabPanel>
            <p>This is a fantastic product that you will absolutely love.</p>
          </TabPanel>
          <TabPanel>
            <ul>
              <li>Weight: 10kg</li>
              <li>Dimensions: 10x20x30 cm</li>
            </ul>
          </TabPanel>
          <TabPanel>
            <p>No reviews yet. Be the first!</p>
          </TabPanel>
        </TabPanels>
      </Tabs>
    </div>
  );
}
```

**Interactive Behavior**:

- The component renders with "Tab 1" active and its content visible.
- Clicking on "Tab 2" makes it active, highlights its title, and shows the content for tab 2.

This pattern is extremely powerful. The `ProductInfo` component doesn't need to manage any state. It just declaratively describes the structure of the tabs. All the complex state management is hidden inside our reusable compound components.

## Production Perspective

- **API Design**: The primary goal of this pattern is to create a good developer experience. By breaking a complex component into smaller, cooperative pieces, you make the API more flexible and easier to understand.
- **Encapsulation**: The context is internal to the set of compound components. The consumer of the `Tabs` component doesn't need to know or care that context is being used.
- **Flexibility**: This pattern allows for great flexibility. The consumer can rearrange the `TabList` and `TabPanels` or insert other elements between them, and it will still work, because the communication happens via the shared context, not direct parent-child relationships.
- **Accessibility**: When building compound components like tabs, accordions, or menus, it's crucial to implement the appropriate ARIA attributes to make them accessible to screen readers. This logic can be encapsulated within the components themselves.

## Controlled vs Uncontrolled Components Deep Dive

## Learning Objective

Distinguish between controlled and uncontrolled components, particularly for form inputs, and understand how React Actions relate to this concept.

## Why This Matters

How you manage data in form inputs is a fundamental architectural decision in React. "Controlled" components put React in charge of the form's state, while "uncontrolled" components let the DOM handle it. Understanding the trade-offs is key to building forms, and the new Actions paradigm introduces a modern take on the uncontrolled approach.

## Discovery Phase: A Direct Comparison

Let's look at the two patterns side-by-side for a simple text input.

### Controlled Component

In a controlled component, the value of the input is "controlled" by React state. The state is the single source of truth.

```jsx
"use client";
import { useState } from "react";

function ControlledInput() {
  const [value, setValue] = useState("Hello");

  const handleChange = (e) => {
    setValue(e.target.value);
  };

  return (
    <div>
      <h4>Controlled</h4>
      <input type="text" value={value} onChange={handleChange} />
      <p>Current value: {value}</p>
    </div>
  );
}
```

**Behavior**:

- The input displays "Hello".
- Every time you type, `handleChange` fires, `setValue` is called, and the component re-renders. The input's `value` is always driven by the `value` state variable.

### Uncontrolled Component

In an uncontrolled component, the DOM manages the input's state internally. We use a `ref` to read the value from the DOM when we need it, typically on form submission.

```jsx
"use client";
import { useRef } from "react";

function UncontrolledInput() {
  const inputRef = useRef(null);

  const handleSubmit = () => {
    // We only read the value when we need it.
    alert("Submitted value: " + inputRef.current.value);
  };

  return (
    <div>
      <h4>Uncontrolled</h4>
      {/* `defaultValue` sets the initial value, but doesn't control it after that. */}
      <input type="text" defaultValue="Hello" ref={inputRef} />
      <button onClick={handleSubmit}>Submit</button>
    </div>
  );
}
```

**Behavior**:

- The input displays "Hello".
- When you type, React does **not** re-render. The DOM is handling the input's state on its own.
- When you click "Submit", we use the ref to pull the current value directly from the DOM.

## Deep Dive: Trade-offs and the Role of Actions

| Feature             | Controlled Components                                                                                                        | Uncontrolled Components                                                                  |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **Source of Truth** | React state (`useState`)                                                                                                     | The DOM itself                                                                           |
| **Data Flow**       | Value flows from state to input. `onChange` updates state.                                                                   | DOM updates itself. React reads value via `ref`.                                         |
| **Re-renders**      | Re-renders on every keystroke.                                                                                               | Does not re-render on input change.                                                      |
| **Use Cases**       | - Real-time validation<br>- Disabling submit button based on input<br>- Forcing specific formats (e.g., credit card numbers) | - Simple forms<br>- Performance-critical forms<br>- Integrating with non-React libraries |

### How Actions Fit In

The new form Actions model in React 19 is fundamentally an **uncontrolled** pattern. When you use `<form action={...}>`, React automatically gathers the current values from the DOM inputs using the `FormData` API when the form is submitted. Your component does not need to hold the value of every input in its state.

```jsx
// A Server Action
async function searchAction(formData) {
  "use server";
  const query = formData.get("query");
  console.log("Server received search for:", query);
  // ... perform search
}

// The component using the action
function SearchForm() {
  // No useState for the input's value! This is uncontrolled.
  return (
    <form action={searchAction}>
      <input name="query" type="search" defaultValue="Initial query" />
      <button type="submit">Search</button>
    </form>
  );
}
```

This is powerful because it's simple and performant. For many forms, you don't need to know what the user is typing on every single keystroke. You only care about the final value on submission.

### Creating "Controlled-like" Behavior with Actions

What if you need both? You want the simplicity of actions, but you also need to show real-time validation. You can combine the patterns. You can use `useState` to control the input for the user's immediate feedback, but still rely on the action for the submission logic.

```jsx
"use client";
import { useActionState } from "react";
import { useState } from "react";

async function signupAction(prevState, formData) {
  /* ... */
}

function SignupForm() {
  const [actionState, formAction] = useActionState(signupAction, {});
  const [username, setUsername] = useState("");

  const isUsernameValid = username.length >= 3;

  return (
    <form action={formAction}>
      <label>Username:</label>
      {/* This input is CONTROLLED by our component's state */}
      <input
        name="username"
        value={username}
        onChange={(e) => setUsername(e.target.value)}
      />
      {!isUsernameValid && (
        <p style={{ color: "red" }}>Username must be at least 3 characters.</p>
      )}

      <button type="submit" disabled={!isUsernameValid}>
        Sign Up
      </button>
    </form>
  );
}
```

This hybrid approach gives you the best of both worlds: instant client-side feedback from a controlled input, and robust, progressive submission handling from an uncontrolled form action.

## Production Perspective

- **Default to Uncontrolled/Actions**: For new forms in React 19, start with the uncontrolled action-based approach. It's simpler, more performant, and enables progressive enhancement.
- **Opt-in to Controlled**: Only add the `useState` to control an input when you have a specific feature that requires it, such as real-time validation, formatting as the user types, or synchronizing the input's value with another UI element.
- **Libraries**: Many form libraries (like Formik or React Hook Form) provide their own abstractions over this concept, often giving you a "controller" component that handles the state management for you.

## Prop Drilling and Solutions

## Learning Objective

Identify the problem of "prop drilling" and apply modern solutions, including React Context for client-side state and component composition for server-side data.

## Why This Matters

As applications grow, passing data through many layers of intermediate components becomes cumbersome and makes code hard to maintain. This is "prop drilling." Knowing how to avoid it is crucial for building scalable applications. The RSC architecture provides new ways to think about and solve this old problem.

## Discovery Phase: The Problem Illustrated

Prop drilling is when you pass props through components that don't need them, just to get them to a deeply nested child that does.

```jsx
// The top-level component has the data.
function App() {
  const theme = "dark";
  return <Page theme={theme} />;
}

// Page doesn't use `theme`, it just passes it on.
function Page({ theme }) {
  return <Toolbar theme={theme} />;
}

// Toolbar doesn't use `theme`, it just passes it on.
function Toolbar({ theme }) {
  return <ThemeButton theme={theme} />;
}

// Finally, the component that actually needs the data.
function ThemeButton({ theme }) {
  const style = {
    background: theme === "dark" ? "black" : "white",
    color: theme === "dark" ? "white" : "black",
  };
  return <button style={style}>Themed Button</button>;
}
```

This is painful. If `Toolbar` ever gets refactored to not render `ThemeButton`, the prop chain breaks. If another component needs `theme`, we have to drill it down a different path.

## Deep Dive: Modern Solutions

### Solution 1: React Context (for Client-Side State)

If the data being passed is client-side state or data that many components need (like theme, current user, language), Context is the ideal solution. It allows a parent component to make data available to any component in the tree below it without passing props.

```jsx
"use client";
import { createContext, useContext, useState } from "react";

// 1. Create the context
const ThemeContext = createContext("light");

// The top-level component PROVIDES the value.
export default function App() {
  const [theme, setTheme] = useState("dark");

  return (
    <ThemeContext.Provider value={theme}>
      <Page />
    </ThemeContext.Provider>
  );
}

// Page no longer needs to know about `theme`.
function Page() {
  return <Toolbar />;
}

// Toolbar no longer needs to know about `theme`.
function Toolbar() {
  return <ThemeButton />;
}

// The deeply nested component CONSUMES the value directly.
function ThemeButton() {
  const theme = useContext(ThemeContext); // Or `use(ThemeContext)` in React 19
  const style = {
    /* ... */
  };
  return <button style={style}>Themed Button</button>;
}
```

This breaks the rigid coupling of the prop chain. `ThemeButton` can now be placed anywhere inside the `ThemeContext.Provider` and it will just work.

### Solution 2: Component Composition (for Server-Side Content)

Sometimes, prop drilling isn't about shared state, but about passing specific content down. In the RSC world, you can often solve this by moving the data fetching to the top and passing the rendered component down, rather than the data itself.

**The Problem**: A `Page` needs to pass `user` data to a `Header`.

```jsx
// The "drilling" way
async function Page() {
  const user = await db.users.find(1);
  return <Layout user={user} />; // Layout doesn't need user
}

function Layout({ user, children }) {
  return (
    <div>
      <Header user={user} /> {/* Header needs user */}
      <main>{children}</main>
    </div>
  );
}
```

**The Composition Solution**:
Instead of passing the `user` data through `Layout`, we can render the `Header` in `Page` and pass the _fully rendered component_ to `Layout`.

```jsx
// All of these can be Server Components

async function Header({ user }) {
  return <header>Welcome, {user.name}</header>;
}

function Layout({ header, children }) {
  return (
    <div>
      {header} {/* Renders the component it was given */}
      <main>{children}</main>
    </div>
  );
}

// The top-level component composes the tree.
export default async function Page() {
  const user = await db.users.find(1);

  return (
    <Layout header={<Header user={user} />}>
      <p>This is the main page content.</p>
    </Layout>
  );
}
```

This is a powerful pattern. `Layout` is now more generic and reusable. It doesn't need to know anything about a `user`. It just provides a "slot" for a header. The `Page` component, which is responsible for the page's data, decides what to put in that slot. This solves the prop drilling problem by moving the dependency up to the parent.

## Production Perspective

- **Context for Global State**: Use Context for truly global, client-side concerns that affect many components at different levels: theme, authentication state, language, etc.
- **Composition for Layouts**: Use component composition (`children` and other props that accept JSX) to structure your pages and avoid drilling server-fetched data through layout components.
- **Don't Overuse Context**: Context has a performance cost. When the context value changes, all components that consume it will re-render. Don't use it for high-frequency state updates. Sometimes, drilling a prop one or two levels deep is simpler and more performant than creating a new context for it.

## Component Composition Strategies

## Learning Objective

Master component composition as the primary tool for building flexible and reusable UIs in React, focusing on the "children" prop and creating specialized "slots".

## Why This Matters

"Composition over inheritance" is a core principle of React. Instead of creating complex component hierarchies where components inherit behavior, React encourages you to build small, reusable components and assemble them like building blocks. This approach leads to a more flexible, scalable, and maintainable codebase. In the RSC world, it's also the key to mixing server and client components effectively.

## Discovery Phase: The "children" Prop

The most common and fundamental composition pattern is using the `children` prop. Any JSX you put between a component's opening and closing tags is passed to that component as `props.children`.

This allows us to create generic "box" or "wrapper" components.

```jsx
// A generic wrapper component
function FancyBorder({ children }) {
  return (
    <div style={{ border: "4px solid purple", padding: "1em", margin: "1em" }}>
      {children}
    </div>
  );
}

// We can put anything inside it.
function App() {
  return (
    <FancyBorder>
      <h1>Welcome!</h1>
      <p>This content is inside the fancy border.</p>
    </FancyBorder>
  );
}
```

This is simple but incredibly powerful. `FancyBorder` doesn't need to know or care what it's rendering. It just provides a wrapper. This is the pattern that enables the Client "Shell" / Server "Slot" architecture we've discussed.

## Deep Dive: Creating Multiple "Slots"

What if you need more than one place to inject content? A component isn't limited to just `children`. You can accept any prop that contains JSX. This allows you to create components with multiple named "slots".

Let's build a `SplitPane` layout component.

```jsx
// This component defines two slots: `left` and `right`.
function SplitPane({ left, right }) {
  return (
    <div style={{ display: "flex", gap: "1em" }}>
      <div style={{ flex: 1, border: "1px solid gray" }}>{left}</div>
      <div style={{ flex: 1, border: "1px solid gray" }}>{right}</div>
    </div>
  );
}

// The parent component fills the slots.
function App() {
  return (
    <SplitPane
      left={
        <div>
          <h3>Contacts</h3>
          {/* ... contact list ... */}
        </div>
      }
      right={
        <div>
          <h3>Chat</h3>
          {/* ... chat window ... */}
        </div>
      }
    />
  );
}
```

This pattern is extremely useful for creating reusable page layouts. The `SplitPane` component is completely generic, and the parent `App` component provides the specific content for each slot.

### Specialization through Composition

Another powerful strategy is to create more specialized versions of a component through composition, rather than by adding more props and internal logic.

**The "Bad" Way (using props)**:

```jsx
function Dialog({ type, title, message }) {
  const color = type === 'warning' ? 'orange' : 'blue';
  return (
    <div style={{ border: `2px solid ${color}` }}>
      <h1>{title}</h1>
      <p>{message}</p>
    </div>
  );
}
// Usage: <Dialog type="warning" title="Warning!" message="..." />
```

This component's logic grows more complex as you add more `type`s.

**The "Good" Way (using composition)**:

```jsx

// Start with a generic, composable base component.
function Dialog({ title, children, titleColor }) {
  return (
    <div style={{ border: '2px solid gray' }}>
      <h1 style={{ color: titleColor }}>{title}</h1>
      <div>{children}</div>
    </div>
  );
}

// Create specialized versions by composing the base component.
function WarningDialog(props) {
  return (
    <Dialog titleColor="orange" title={props.title || 'Warning'}>
      {props.children}
    </Dialog>
  );
}

function SuccessDialog(props) {
  return (
    <Dialog titleColor="green" title={props.title || 'Success'}>
      {props.children}
    </Dialog>
  );
}

// Usage is now more semantic and less prone to typos.
function App() {
  return (
    <div>
      <WarningDialog>
        <p>This is a warning message!</p>
      </WarningDialog>
      <SuccessDialog title="It Worked!">
        <p>Your action was successful.</p>
      </SuccessDialog>
    </div>
  );
}
```

This approach is more scalable and maintainable. The base `Dialog` is simple, and the specialized versions (`WarningDialog`, `SuccessDialog`) are easy to understand and create.

## Production Perspective

- **Composition is the Default**: When building React components, your first instinct should always be to solve problems with composition. Before adding a new prop to change a component's behavior, ask yourself if you could achieve the same result by passing in different `children` or creating a specialized wrapper component.
- **Server and Client Harmony**: This pattern is the key to making RSCs and Client Components work together. A Server Component can compose a Client Component shell, passing server-rendered content into its `children` slot. This gives you the best of both worlds: server performance and client interactivity.
- **Design Systems**: Component libraries and design systems are built on these principles. They provide a set of generic, composable "building blocks" (`Box`, `Flex`, `Card`) that developers can assemble to create complex, application-specific UIs.

## When to Split Components

## Learning Objective

Develop a set of practical heuristics for deciding when and how to split a large component into smaller, more manageable ones.

## Why This Matters

Knowing when to break down a component is a critical skill for maintaining a healthy codebase. Large, monolithic components are hard to read, debug, and reuse. Splitting components effectively leads to a more organized, performant, and maintainable application. The introduction of Server and Client components adds a new, important reason to split components.

## Discovery Phase: The Monolithic Component

It's common for components to start small and grow over time as new features are added. This can lead to a "god component" that does too much.

## Module Synthesis 📋

## Module Synthesis: Timeless Principles in a Modern Context

This chapter has been a bridge between the past and future of React architecture. We've explored foundational patterns that have shaped React development for years and re-evaluated them through the powerful new lens of React 19, Server Components, and Actions.

### Key Takeaways

1.  **Principles Endure, Patterns Evolve**: The core principles of separating concerns, composition, and managing state remain the same. However, our tools for implementing them have evolved.
    - **Server Components** are the new "containers" for data fetching.
    - **Custom Hooks** are the new standard for sharing stateful logic, replacing HOCs and Render Props.
    - **Client Components** are the new "presentational" layer for interactive UI.

2.  **Composition is King**: The most powerful and flexible pattern in React is composition, primarily through the `children` prop. This pattern has become even more central in the RSC era, acting as the primary mechanism for embedding static, server-rendered content within interactive client-side layouts.

3.  **The Server/Client Boundary is a Key Architectural Driver**: The decision of where to place the `'use client'` directive is a critical architectural choice. Pushing client boundaries down the tree and splitting components to separate server and client logic are key skills for building performant modern applications.

4.  **Form Patterns are Modernized**: The classic controlled vs. uncontrolled component debate has a new dimension. Form Actions encourage a powerful, uncontrolled pattern by default, while still allowing for controlled inputs when client-side interactivity is needed.

### Looking Forward

We now have a robust mental model for how to structure a modern React application, from the individual component patterns to the high-level server/client architecture. We know how to manage both client-side state and server-side data mutations.

In the next chapter, **Chapter 11: State Management Solutions**, we will zoom in on the challenge of managing state that is truly global to the client-side application. We'll do a deep dive into React's built-in Context API and then explore popular external libraries like Redux and Zustand, providing a clear guide on when and why you might need to reach for a dedicated state management solution.
