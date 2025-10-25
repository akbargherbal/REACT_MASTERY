# Chapter 4: State and Event Handling

## Understanding State in React

## Learning Objective

Differentiate between props and state, and identify when a component needs state to manage its own data over time.

## Why This Matters

If props are how components receive information, state is how they remember things. State is the heart of interactivity in any React application. It's the component's own private memory. Understanding the fundamental difference between props (data from the outside) and state (memory on the inside) is the most critical step to building components that can respond to user interaction.

## Discovery Phase: A Button That Doesn't Remember

Let's build a simple "Like" button component. Our goal is for the button to change its text from "Like" to "Liked!" when clicked. A first attempt might involve trying to use props.

```jsx
// This component does NOT work as intended.

function LikeButton({ isLiked }) {
  let buttonText = isLiked ? "Liked!" : "Like";

  const handleClick = () => {
    // We want to change `isLiked` to true here, but how?
    // Props are read-only, so we can't do `isLiked = true;`
    console.log("Button clicked, but nothing will change visually.");
  };

  return <button onClick={handleClick}>{buttonText}</button>;
}

// Usage in App.jsx
// <LikeButton isLiked={false} />
```

**Interactive Behavior**:

```
[Like]
(Click)
→ [Like] (The text does not change)
```

This component has a problem: it has no memory. When `handleClick` is called, it has no way to "remember" that it was clicked for the next render. We can't change the `isLiked` prop because props are read-only; they are controlled by the parent component.

To solve this, the component needs its own internal memory. This is what **state** is for.

## Deep Dive: Props vs. State

Let's formally define the two.

| Concept        | Props (Properties)                                    | State                                             |
| :------------- | :---------------------------------------------------- | :------------------------------------------------ |
| **Purpose**    | To pass data from a parent to a child.                | To manage a component's own internal data.        |
| **Ownership**  | Owned and controlled by the **parent** component.     | Owned and controlled by the **component itself**. |
| **Mutability** | **Read-only**. A component must not change its props. | **Mutable**. Can be updated by the component.     |
| **Analogy**    | Arguments passed to a function.                       | Variables declared inside a function.             |

**A simple analogy**:

- **Props** are like your name and date of birth on your driver's license. They are given to you by an external authority (the government/parent component) and you can't change them yourself.
- **State** is like your current mood or whether you are hungry. It's internal to you, it changes over time based on events, and you manage it yourself.

A component needs state when some data associated with it changes over time in response to user interaction or other events. Ask yourself: "Does this data need to be remembered by the component across renders?" If the answer is yes, it needs to be state.

## The useState Hook

## Learning Objective

Use the `useState` hook to add a state variable to a function component and update it with the setter function.

## Why This Matters

The `useState` hook is the fundamental tool for adding state to your components. It's the most common and important React hook you will learn. Mastering `useState` is the key that unlocks your ability to build dynamic, interactive user interfaces.

## Discovery Phase: Fixing the Like Button

Let's fix our broken `LikeButton` from the previous section by giving it state. To do this, we'll use the `useState` hook.

```jsx
import { useState } from "react";

function LikeButton() {
  // 1. Call useState to create a state variable.
  // It returns an array with two elements:
  // - The current state value (`isLiked`)
  // - A function to update it (`setIsLiked`)
  const [isLiked, setIsLiked] = useState(false); // `false` is the initial state.

  const handleClick = () => {
    // 2. Call the setter function to update the state.
    setIsLiked(true);
  };

  // 3. The UI is now driven by the state variable.
  return <button onClick={handleClick}>{isLiked ? "Liked!" : "Like"}</button>;
}
```

**Interactive Behavior**:

```
[Like]
(Click)
→ [Liked!] (The text now changes!)
```

This works! Let's trace exactly what happens.

## Deep Dive: The Anatomy of `useState`

### The Render Cycle with State

1.  **Initial Render**:
    - `LikeButton` is called for the first time.
    - `useState(false)` is called. React initializes this piece of state to `false` and returns `[false, function]`.
    - Our `isLiked` variable is `false`.
    - The component returns `<button>Like</button>`. React updates the DOM.

2.  **User Interaction**:
    - The user clicks the button.
    - The `onClick` event fires, calling our `handleClick` function.

3.  **State Update**:
    - Inside `handleClick`, we call `setIsLiked(true)`.
    - Calling this setter function does two things:
      1.  It tells React to update the state value for the next render to `true`.
      2.  It tells React to **schedule a re-render** of this component.

4.  **Re-render**:
    - React calls the `LikeButton` function again.
    - This time, when `useState(false)` is encountered, React remembers the current value for this state is `true`, so it returns `[true, function]`.
    - Our `isLiked` variable is now `true`.
    - The component returns `<button>Liked!</button>`.

5.  **DOM Update (Reconciliation)**:
    - React compares the new render output (`<button>Liked!</button>`) with the old one (`<button>Like</button>`).
    - It sees that the text inside the button has changed and efficiently updates only that part of the real DOM.

### Common Confusion: State Updates Seem Asynchronous

**You might think**: Calling `setCount(count + 1)` immediately changes the `count` variable.

**Actually**: State setter functions schedule an update. The state variable itself will only have the new value on the _next render_.

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    // This schedules a re-render with count = 1
    setCount(count + 1);

    // 🛑 This will log 0, not 1!
    // The `count` variable in *this* render is still 0.
    console.log(count);
  };
  // ...
}
```

**How to remember**: Think of `setState` as "requesting a re-render with this new value," not "change this variable right now."

### Updating State Based on the Previous State

What happens if you need to update state multiple times in quick succession, like in a single event handler?

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleTripleClick = () => {
    setCount(count + 1); // With count=0, this requests a render with 1
    setCount(count + 1); // With count=0, this also requests a render with 1
    setCount(count + 1); // With count=0, this also requests a render with 1
  };
  // Clicking this button will only increment the count by 1, not 3!
  return <button onClick={handleTripleClick}>+3</button>;
}
```

To fix this, you can pass an **updater function** to the state setter. This function receives the pending state and should return the new state. React queues these functions and runs them in order.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleTripleClick = () => {
    // Pass a function to guarantee you're working with the latest state.
    setCount((prevCount) => prevCount + 1);
    setCount((prevCount) => prevCount + 1);
    setCount((prevCount) => prevCount + 1);
  };
  // Now this works correctly!
  return <button onClick={handleTripleClick}>+3</button>;
}
```

**Rule of thumb**: If your new state depends on the previous state, always use the updater function form (`setSomething(prev => ...)`) to avoid bugs from stale state.

## Event Handling in React

## Learning Objective

Attach event handlers like `onClick` and `onChange` to JSX elements to trigger state updates and other logic.

## Why This Matters

Interactivity is all about responding to user actions. Event handlers are the bridge between the user's actions (clicking, typing, hovering) and your component's JavaScript logic. Understanding how to wire them up correctly is essential for making your application do anything useful.

## Discovery Phase: Responding to a Click

The simplest event is a click. In HTML, you might use an `onclick` attribute with a string of JavaScript. In React, you pass a function to a camelCased prop like `onClick`.

```jsx
// Version 1: Inline Arrow Function
// Good for very short, simple actions.
function AlertButton() {
  return <button onClick={() => alert("You clicked me!")}>Click Me</button>;
}

// Version 2: Separate Handler Function
// Better for readability and more complex logic.
function LogButton() {
  const handleClick = () => {
    console.log("Button was clicked at", new Date());
  };

  return <button onClick={handleClick}>Log Click</button>;
}
```

Notice in `LogButton`, we pass `onClick={handleClick}`. We are passing a _reference_ to the `handleClick` function. We are **not** calling it, like `onClick={handleClick()}`.

### Common Confusion: `onClick={handleClick}` vs. `onClick={handleClick()}`

**You might think**: I need to call the function, so I should use parentheses.

**Actually**: `onClick={handleClick()}` would call the `handleClick` function _during the render process_ and pass its return value (which is `undefined`) to `onClick`. This would cause your function to fire every time the component renders, not just on click.

**How to remember**: You are telling React, "When a click happens in the future, please call this function for me." You are giving it a recipe, not the finished meal.

## Deep Dive: Common Event Handling Patterns

### Passing Arguments to Event Handlers

What if you need to pass an ID or some other data to your handler? You can't just write `onClick={handleDelete(item.id)}` for the reason we just discussed. The solution is to use a wrapper arrow function.

```jsx
function ItemList({ items }) {
  const handleDelete = (id, name) => {
    alert(`Deleting item ${id}: ${name}`);
  };

  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>
          {item.name}
          {/*
            This inline arrow function is NOT called during render.
            It's just defined. When the button is clicked,
            this arrow function is executed, which then calls handleDelete.
          */}
          <button onClick={() => handleDelete(item.id, item.name)}>
            Delete
          </button>
        </li>
      ))}
    </ul>
  );
}
```

### Reading from the Event Object

For events like `onChange` on an input, you often need to get information from the event itself. React passes a "synthetic event" object to your handler, which works similarly to the native browser event. The most common use case is getting the current value of an input with `event.target.value`.

```jsx
import { useState } from "react";

function SearchBar() {
  const [query, setQuery] = useState("");

  // The event object `e` is automatically passed to the handler.
  const handleChange = (e) => {
    // e.target refers to the DOM element that triggered the event (the input).
    // e.target.value holds its current value.
    setQuery(e.target.value);
  };

  return (
    <div>
      <input
        type="text"
        placeholder="Search..."
        value={query}
        onChange={handleChange}
      />
      <p>Current query: {query}</p>
    </div>
  );
}
```

This example combines state and event handling to create what's known as a **controlled component**, which is our next topic.

## Controlled vs Uncontrolled Components

## Learning Objective

Differentiate between controlled and uncontrolled form inputs and understand the trade-offs of each approach.

## Why This Matters

This is a fundamental pattern for handling forms in React. Choosing the right approach is key to building forms that are predictable, easy to validate, and simple to integrate with the rest of your application's state. The "controlled" approach is the most common and powerful pattern in the React ecosystem.

## Discovery Phase: Two Ways to Get an Input's Value

Imagine a simple form where you want to log the user's name on submit. How do we get the text from the `<input>`?

### Approach 1: Uncontrolled (Let the DOM Handle It)

In an uncontrolled component, the DOM is the "source of truth" for the input's value. We don't tell the input what its value should be. We only ask for it when we need it, typically using a `ref`.

```jsx
import { useRef } from "react";

function UncontrolledNameForm() {
  // Create a ref to hold a reference to the input DOM node.
  const inputRef = useRef(null);

  const handleSubmit = (e) => {
    e.preventDefault();
    // We read the value directly from the DOM node when the form is submitted.
    alert(`Hello, ${inputRef.current.value}`);
  };

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Name:
        {/* React will put the input's DOM element into `inputRef.current` */}
        <input type="text" ref={inputRef} />
      </label>
      <button type="submit">Submit</button>
    </form>
  );
}
```

This is simple and feels like traditional HTML/JavaScript. React isn't "in charge" of the input's value.

### Approach 2: Controlled (Let React Handle It)

In a controlled component, React state is the "single source of truth." We use state to hold the input's value and update it on every keystroke.

```jsx
import { useState } from "react";

function ControlledNameForm() {
  // 1. Create a piece of state to hold the input's value.
  const [name, setName] = useState("");

  const handleSubmit = (e) => {
    e.preventDefault();
    // We read the value directly from our state.
    alert(`Hello, ${name}`);
  };

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Name:
        <input
          type="text"
          // 2. The input's value is driven by React state.
          value={name}
          // 3. On every change, we update the React state.
          onChange={(e) => setName(e.target.value)}
        />
      </label>
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Deep Dive: A Direct Comparison

The flow of data is completely different:

- **Uncontrolled**: User types -> DOM updates input -> On submit, React reads from DOM.
- **Controlled**: User types -> `onChange` fires -> `setName` updates state -> React re-renders -> Input `value` prop is updated from state.

| Feature              | Uncontrolled Components                               | Controlled Components                                      |
| :------------------- | :---------------------------------------------------- | :--------------------------------------------------------- |
| **Source of Truth**  | The DOM                                               | React state                                                |
| **Data Flow**        | One-way (read from DOM on demand).                    | Two-way (state updates UI, UI updates state).              |
| **Typical Use Case** | Very simple forms, file inputs.                       | Most forms, especially with validation or dynamic UI.      |
| **Pros**             | Simpler setup, less code for basic forms.             | Enables instant validation, conditional logic, formatting. |
| **Cons**             | Harder to implement features like instant validation. | More boilerplate code for simple cases.                    |

### Production Perspective

**Professionals almost always prefer controlled components.** Why? Because modern applications require more than just getting a value on submit.

With a controlled component, you can:

- **Perform instant validation**: Show an error message as soon as the user types an invalid character.
- **Conditionally disable the submit button**: ` <button disabled={name.length === 0}>Submit</button>`.
- **Enforce a specific format**: Automatically add dashes to a credit card number or parentheses to a phone number as the user types.
- **Synchronize multiple inputs**: Imagine a "price" slider that also updates a text input field.

While uncontrolled components have their place for very simple, fire-and-forget forms, the controlled pattern gives you the power and flexibility needed for building rich, interactive user experiences.

## Forms and Input Handling

## Learning Objective

Build a complete form with multiple inputs, manage their state in a single object, and handle form submission.

## Why This Matters

Most real-world forms have more than one field. Managing the state for each field individually can become tedious. Learning how to manage the entire form's state in a single object with a generic handler function is a scalable pattern you will use in almost every project.

## Discovery Phase: The Repetitive Approach

Let's build a registration form. A naive approach would be to create a separate `useState` for each field.

```jsx
import { useState } from "react";

function RegistrationFormNaive() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");

  const handleEmailChange = (e) => setEmail(e.target.value);
  const handlePasswordChange = (e) => setPassword(e.target.value);

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log({ email, password });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="email" value={email} onChange={handleEmailChange} />
      <input type="password" value={password} onChange={handlePasswordChange} />
      <button type="submit">Register</button>
    </form>
  );
}
```

This works, but imagine a form with ten fields. You'd have ten state variables and ten handler functions. This is not maintainable.

## Deep Dive: The Single State Object Pattern

A much better approach is to hold the entire form's data in a single state object.

```jsx
import { useState } from "react";

function RegistrationForm() {
  // 1. Use a single state object to hold all form data.
  const [formData, setFormData] = useState({
    email: "",
    password: "",
  });

  // 2. Create a single, generic change handler.
  const handleChange = (e) => {
    // Destructure the name and value from the input element.
    const { name, value } = e.target;

    setFormData((prevFormData) => ({
      // Copy the old fields
      ...prevFormData,
      // And update the one that changed.
      // The `[name]` syntax is a "computed property name".
      // If `name` is "email", this becomes { email: value }.
      [name]: value,
    }));
  };

  const handleSubmit = (e) => {
    // 3. Prevent the default browser form submission (full page reload).
    e.preventDefault();
    console.log("Form submitted with data:", formData);
    alert(`Registered with email: ${formData.email}`);
  };

  return (
    // 4. Use the onSubmit handler on the <form> element.
    <form onSubmit={handleSubmit}>
      <label>Email</label>
      <input
        type="email"
        // The `name` attribute is now crucial! It must match the key in our state object.
        name="email"
        value={formData.email}
        onChange={handleChange}
      />
      <br />
      <label>Password</label>
      <input
        type="password"
        name="password"
        value={formData.password}
        onChange={handleChange}
      />
      <br />
      <button type="submit">Register</button>
    </form>
  );
}
```

This pattern is far more scalable. To add a new `username` field, you only need to:

1.  Add `username: ''` to the initial state object.
2.  Add the new `<input name="username" ... />` to your JSX.

The `handleChange` function doesn't need to be touched. The key is the `name` attribute on each input, which tells the handler which property of the state object to update.

## State Management Best Practices

## Learning Objective

Apply principles for structuring and updating state effectively to avoid bugs and improve maintainability.

## Why This Matters

As your components grow, the complexity of their state can become a major source of bugs. Following a few simple principles for how you structure and update your state will make your components more predictable, easier to debug, and simpler to reason about.

## Discovery Phase: The Impossible State Problem

Imagine you're fetching data. You might model your component's state with several booleans.

```jsx
// Anti-pattern: This state structure can lead to bugs.
function DataFetcher() {
  const [isLoading, setIsLoading] = useState(true);
  const [isError, setIsError] = useState(false);
  const [data, setData] = useState(null);

  // ... data fetching logic that sets these booleans ...

  // What if a bug causes `isLoading` and `isError` to both be true?
  // Our UI would be in a confusing, contradictory state.
  if (isLoading) return <p>Loading...</p>;
  if (isError) return <p>Error!</p>;
  return <div>{data}</div>;
}
```

The problem here is that our state allows for "impossible" situations. A request can't be both loading and in an error state at the same time. This structure makes it easy to write bugs where you forget to reset one boolean when setting another.

## Deep Dive: Four Principles of Good State

### Principle 1: Group Related State

If two or more state variables always change together, consider grouping them into a single state variable. For our data fetching example, we can use a single `status` string.

```jsx
// Good: Impossible states are now impossible.
function DataFetcher() {
  // The status can only be one of these values at a time.
  const [status, setStatus] = useState("loading"); // 'loading', 'success', 'error'
  const [data, setData] = useState(null);

  // ... fetch logic now calls `setStatus('success')` or `setStatus('error')` ...

  if (status === "loading") return <p>Loading...</p>;
  if (status === "error") return <p>Error!</p>;
  return <div>{data}</div>;
}
```

### Principle 2: Avoid Redundant State

Do not store data in state if it can be calculated from existing props or state during render. Storing it creates two sources of truth that can fall out of sync.

```jsx
function NameForm() {
  const [firstName, setFirstName] = useState("");
  const [lastName, setLastName] = useState("");

  // 🛑 Anti-pattern: `fullName` is redundant state.
  // const [fullName, setFullName] = useState('');
  // useEffect(() => {
  //   setFullName(firstName + ' ' + lastName);
  // }, [firstName, lastName]);

  // ✅ Good: Calculate it on the fly during render. It's always up to date.
  const fullName = `${firstName} ${lastName}`;

  return (
    <div>
      <input value={firstName} onChange={(e) => setFirstName(e.target.value)} />
      <input value={lastName} onChange={(e) => setLastName(e.target.value)} />
      <p>Full Name: {fullName}</p>
    </div>
  );
}
```

### Principle 3: Treat State as Immutable

As we've seen, you must never modify state objects or arrays directly. Always create a new one. React relies on reference equality (`===`) to detect changes. If you mutate an object, the reference doesn't change, and React won't re-render.

```jsx
// 🛑 BAD: Mutating state
const handleAddItem = () => {
  // This pushes to the original array. The reference doesn't change.
  // React will not detect this as a state change.
  todos.push({ id: 4, text: "New Todo" });
  setTodos(todos);
};

// ✅ GOOD: Creating a new array
const handleAddItem = () => {
  const newTodo = { id: 4, text: "New Todo" };
  // The spread operator creates a new array with the new item.
  setTodos([...todos, newTodo]);
};
```

### Principle 4: Keep State as Flat as Possible

Deeply nested state objects can be cumbersome to update immutably.
`setFormData({...formData, user: {...formData.user, address: {...}}})`
If your state becomes this nested, it's often a sign that you should either:
a) Flatten your state structure.
b) Split your component into smaller components, each managing its own piece of state.
c) Use a more powerful state management hook like `useReducer` (covered in Chapter 6), which is designed for complex state transitions.

## Lifting State Up

## Learning Objective

Share state between sibling components by moving it to their closest common ancestor.

## Why This Matters

This is the primary, built-in React pattern for component communication. Often, two child components need to share or react to the same data. "Lifting state up" is the mechanism that allows you to keep them in sync. It's the solution to the question, "How do I make component A talk to component B?"

## Discovery Phase: Components Out of Sync

Imagine a temperature converter with two input fields: one for Celsius and one for Fahrenheit. When you type in one, the other should update. If each component manages its own temperature state, they have no way of knowing about each other.

```jsx
// This does NOT work. The inputs are not connected.
function TemperatureInput({ scale }) {
  const [temperature, setTemperature] = useState("");
  return (
    <fieldset>
      <legend>Enter temperature in {scale}:</legend>
      <input
        value={temperature}
        onChange={(e) => setTemperature(e.target.value)}
      />
    </fieldset>
  );
}

function Calculator() {
  return (
    <div>
      <TemperatureInput scale="Celsius" />
      <TemperatureInput scale="Fahrenheit" />
    </div>
  );
}
```

Typing in the Celsius input only updates its own internal `temperature` state. The Fahrenheit input knows nothing about it. They are out of sync.

## Deep Dive: The Lifting State Pattern

To fix this, we need a single "source of truth" for the temperature. The state needs to live in a place that can be accessed by both inputs. That place is their closest common ancestor: the `Calculator` component.

The pattern involves three steps:

1.  **Move the state** from the children up to the common parent.
2.  **Pass the state down** to the children as props.
3.  **Pass event handlers down** from the parent to the children as props, so the children can notify the parent when the state needs to change.

**Data flows down. Events flow up.**

Let's refactor our `Calculator`.

```jsx
import { useState } from "react";

// Helper functions for conversion
function toCelsius(fahrenheit) {
  return ((fahrenheit - 32) * 5) / 9;
}
function toFahrenheit(celsius) {
  return (celsius * 9) / 5 + 32;
}

// The child component is now "dumber". It just renders props and calls a function.
function TemperatureInput({ scale, temperature, onTemperatureChange }) {
  return (
    <fieldset>
      <legend>Enter temperature in {scale}:</legend>
      <input
        value={temperature}
        onChange={(e) => onTemperatureChange(e.target.value)}
      />
    </fieldset>
  );
}

// The parent component is now "smarter". It owns the state.
function Calculator() {
  // 1. State is lifted up to the parent.
  const [temperature, setTemperature] = useState("");
  const [scale, setScale] = useState("c"); // 'c' for Celsius, 'f' for Fahrenheit

  // 3. Parent defines the handlers that update its own state.
  const handleCelsiusChange = (temp) => {
    setScale("c");
    setTemperature(temp);
  };

  const handleFahrenheitChange = (temp) => {
    setScale("f");
    setTemperature(temp);
  };

  // Calculate the "other" value based on the current state.
  const celsius = scale === "f" ? toCelsius(temperature) : temperature;
  const fahrenheit = scale === "c" ? toFahrenheit(temperature) : temperature;

  return (
    <div>
      {/* 2. Parent passes state and handlers down as props. */}
      <TemperatureInput
        scale="Celsius"
        temperature={celsius}
        onTemperatureChange={handleCelsiusChange}
      />
      <TemperatureInput
        scale="Fahrenheit"
        temperature={fahrenheit}
        onTemperatureChange={handleFahrenheitChange}
      />
    </div>
  );
}
```

**Let's trace the flow when you type "100" into the Celsius input:**

1.  The Celsius `TemperatureInput`'s `onChange` event fires.
2.  It calls its `onTemperatureChange` prop, which is the `handleCelsiusChange` function from the parent, with the value "100".
3.  `handleCelsiusChange` in `Calculator` is executed.
4.  It calls `setScale('c')` and `setTemperature('100')`.
5.  This triggers a re-render of `Calculator`.
6.  During the re-render, `celsius` is calculated as `100` and `fahrenheit` is calculated as `toFahrenheit(100)`, which is `212`.
7.  `Calculator` renders both `TemperatureInput` children, passing down the new values:
    - The Celsius input receives `temperature="100"`.
    - The Fahrenheit input receives `temperature="212"`.
8.  Both inputs are now in sync, with the `Calculator` component as the single source of truth.

### Production Perspective

Lifting state up is the canonical way to share state in React. However, if you have to pass state through many layers of intermediate components that don't use the state themselves, it can become cumbersome. This is known as **prop drilling**. For these more complex scenarios, React provides the Context API (Chapter 6) and the ecosystem provides state management libraries like Zustand or Redux (Chapter 11). But for communication between siblings or close relatives, lifting state up is the correct and idiomatic solution.

## Building Interactive UIs

## Learning Objective

Synthesize all concepts from the chapter to build a small, complete, interactive application: a Todo list.

## Why This Matters

This section is where all the theory comes together. By building a familiar application from scratch, you will solidify your understanding of components, props, state, event handling, and lifting state up. This practical application will bridge the gap between knowing the concepts and knowing how to apply them to solve a real problem.

## Discovery Phase: Todo List Requirements

Let's build a simple Todo list application with the following features:

1.  An input field and a button to add new todos.
2.  A list of all current todos.
3.  The ability to mark a todo as complete by clicking it.
4.  The ability to delete a todo.

First, let's think about the component structure:

- `TodoApp`: The main container component that will hold the application state.
- `TodoForm`: The form for adding new todos.
- `TodoList`: The component that renders the list of todos.
- `TodoItem`: A single todo item, with its own toggle and delete buttons.

Where should the state live? The list of todos needs to be accessed by `TodoList` (to display it) and modified by `TodoForm` (to add to it) and `TodoItem` (to toggle/delete from it). The closest common ancestor is `TodoApp`. Therefore, we will lift the `todos` state up to `TodoApp`.

## Deep Dive: Building the Todo App

Here is the complete code for the application, broken down by component.

```jsx
import React, { useState } from "react";

// --- Child Component: TodoItem ---
// Receives a todo object and handler functions from its parent.
function TodoItem({ todo, onToggle, onDelete }) {
  return (
    <li style={{ textDecoration: todo.completed ? "line-through" : "none" }}>
      <span onClick={() => onToggle(todo.id)}>{todo.text}</span>
      <button onClick={() => onDelete(todo.id)} style={{ marginLeft: "10px" }}>
        &times;
      </button>
    </li>
  );
}

// --- Child Component: TodoList ---
// Receives the list of todos and handlers, and maps over them to render TodoItems.
function TodoList({ todos, onToggle, onDelete }) {
  return (
    <ul>
      {todos.map((todo) => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={onToggle}
          onDelete={onDelete}
        />
      ))}
    </ul>
  );
}

// --- Child Component: TodoForm ---
// Manages its own state for the input field. Calls a parent function on submit.
function TodoForm({ onAddTodo }) {
  const [newTodoText, setNewTodoText] = useState("");

  const handleSubmit = (e) => {
    e.preventDefault();
    if (newTodoText.trim() === "") return; // Don't add empty todos
    onAddTodo(newTodoText);
    setNewTodoText(""); // Clear input after submission
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={newTodoText}
        onChange={(e) => setNewTodoText(e.target.value)}
        placeholder="What needs to be done?"
      />
      <button type="submit">Add Todo</button>
    </form>
  );
}

// --- Parent Component: TodoApp ---
// Owns the state and the logic for manipulating it.
export default function TodoApp() {
  const [todos, setTodos] = useState([
    { id: 1, text: "Learn React", completed: true },
    { id: 2, text: "Build a Todo App", completed: false },
  ]);

  const handleAddTodo = (text) => {
    const newTodo = {
      id: Date.now(), // A simple way to get a unique ID
      text: text,
      completed: false,
    };
    setTodos((prevTodos) => [...prevTodos, newTodo]);
  };

  const handleToggle = (id) => {
    setTodos((prevTodos) =>
      prevTodos.map((todo) =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo,
      ),
    );
  };

  const handleDelete = (id) => {
    setTodos((prevTodos) => prevTodos.filter((todo) => todo.id !== id));
  };

  return (
    <div>
      <h1>Todo List</h1>
      <TodoForm onAddTodo={handleAddTodo} />
      <TodoList todos={todos} onToggle={handleToggle} onDelete={handleDelete} />
    </div>
  );
}
```

### Tracing the Data Flow for "Delete"

Let's trace what happens when you click the delete button (`&times;`) for the "Learn React" todo:

1.  The `onClick` handler in the `TodoItem` component is triggered.
2.  It calls its `onDelete` prop, passing up its own `todo.id` (which is `1`).
3.  The `onDelete` prop in `TodoItem` was passed down from `TodoList`, which in turn received it from `TodoApp`. So, the `handleDelete` function in `TodoApp` is executed with the argument `1`.
4.  `handleDelete` calls `setTodos`, passing an updater function.
5.  The updater function receives the current `todos` array and filters it, returning a new array that no longer contains the todo with `id: 1`.
6.  React schedules a re-render of `TodoApp` with the new, shorter `todos` array.
7.  `TodoApp` re-renders `TodoList`, passing the new array as a prop.
8.  `TodoList` re-renders and maps over the new array. It no longer creates a `TodoItem` for the deleted todo.
9.  React updates the DOM to remove the corresponding `<li>` element.

This example perfectly illustrates the "state flows down, events flow up" architecture that is central to building applications in React.

## Module Synthesis 📋

## Module 4 Synthesis

In this pivotal chapter, we brought our components to life with interactivity. We established the crucial distinction between **props** (read-only data from a parent) and **state** (a component's internal, mutable memory). We learned that state is what a component needs when it must "remember" something that changes over time.

Our primary tool for this was the **`useState` hook**, which provides a state variable and a setter function to schedule re-renders. We saw how to connect user actions to our logic using **event handlers** like `onClick` and `onChange`, and learned the correct way to pass arguments to them.

We dove deep into the world of forms, contrasting **uncontrolled components** (where the DOM is the source of truth) with the more powerful and idiomatic **controlled components** (where React state is the source of truth). We mastered the scalable pattern of managing form state in a single object with a generic `handleChange` function.

Finally, we tied it all together with the most important pattern for component communication: **lifting state up**. By moving state to the closest common ancestor, we enabled sibling components to stay in sync, creating a unidirectional data flow where "state flows down, and events flow up." The final Todo list project demonstrated how all these concepts synthesize to create a complete, interactive application.

## Looking Ahead to Chapter 5

Our components are now interactive, but they exist in a bubble. They don't yet know how to interact with the "outside world"—for example, how to fetch data from a server when they first appear, or how to clean up a connection when they disappear. In the next chapter, **Component Lifecycle and Effects**, we will introduce the `useEffect` hook. This will allow us to synchronize our components with external systems, manage side effects, and gain full control over the component's lifecycle.
