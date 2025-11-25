# Chapter 3: State and Events

## useState: Local Component State

## The Heart of Interactivity: State

So far, our components have been static. They receive props and render UI, but they don't react to user input. They are like photographs: fixed and unchanging. To build applications, we need our components to be like movies: dynamic and responsive.

The core concept that enables this interactivity in React is **state**.

**State is any data that a component "remembers" over time and that can change as a result of user interaction.** When a component's state changes, React automatically re-renders the component to reflect that new state. This is the fundamental loop of a React application:

1.  User interacts with the UI (e.g., clicks a button, types in a field).
2.  An event handler is triggered.
3.  The event handler updates the component's state.
4.  React detects the state change and re-renders the component.
5.  The UI now shows the new state.

To manage state within a function component, we use a special function provided by React called a **Hook**. The most fundamental hook is `useState`.

### Phase 1: Establish the Reference Implementation

Let's build a simple, realistic component that we will make interactive throughout this chapter. This will be our **anchor example**.

**The Goal**: A newsletter signup form. It needs an input field for an email address and a "Subscribe" button.

First, let's create the static, non-functional version.

**Project Structure**:
```
src/
└── components/
    └── NewsletterForm.tsx   ← Our new component
```

Here is the initial, purely presentational code. It does nothing yet.

```tsx
// src/components/NewsletterForm.tsx

function NewsletterForm() {
  return (
    <div style={{ border: '1px solid #ccc', padding: '16px', borderRadius: '8px', maxWidth: '400px' }}>
      <h2>Subscribe to our Newsletter</h2>
      <p>Enter your email to get the latest updates.</p>
      <input
        type="email"
        placeholder="you@example.com"
        style={{ width: 'calc(100% - 10px)', padding: '8px', marginBottom: '8px' }}
      />
      <button style={{ width: '100%', padding: '10px', background: '#0070f3', color: 'white', border: 'none', borderRadius: '4px' }}>
        Subscribe
      </button>
    </div>
  );
}

export default NewsletterForm;
```

This component renders, but it's a dead end. Typing in the input field or clicking the button has no effect. Our goal is to bring it to life.

### Iteration 1: The Wrong Way - Using a Regular Variable

A developer new to React might think: "I need to remember what the user types. I'll just use a JavaScript variable." Let's see why this approach fails.

We'll add a variable `email` and try to display its value below the form. We'll also add a (non-working) `handleChange` function to update it.

```tsx
// src/components/NewsletterForm.tsx (Attempt 1 - This will NOT work)

function NewsletterForm() {
  let email = ''; // A regular JavaScript variable

  const handleChange = (event) => {
    // This function will be called, but React won't care
    email = event.target.value;
    console.log('Variable updated:', email);
  };

  return (
    <div style={{ border: '1px solid #ccc', padding: '16px', borderRadius: '8px', maxWidth: '400px' }}>
      <h2>Subscribe to our Newsletter</h2>
      <p>Enter your email to get the latest updates.</p>
      <input
        type="email"
        placeholder="you@example.com"
        onChange={handleChange} // We've attached an event listener
        style={{ width: 'calc(100% - 10px)', padding: '8px', marginBottom: '8px' }}
      />
      <button style={{ width: '100%', padding: '10px', background: '#0070f3', color: 'white', border: 'none', borderRadius: '4px' }}>
        Subscribe
      </button>
      <p style={{ marginTop: '16px' }}>Current email: {email}</p>
    </div>
  );
}

export default NewsletterForm;
```

Let's run this code and analyze the failure.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
When you type into the input field, the text "Current email:" on the screen remains blank. The input field itself shows the typed text, but our component's display of the `email` variable does not update.

**Browser Console Output**:
```
Variable updated: t
Variable updated: te
Variable updated: tes
Variable updated: test
Variable updated: test@
Variable updated: test@e
Variable updated: test@ex
...
```
The `console.log` shows that our `handleChange` function is running and the `email` variable *is* being updated in memory.

**React DevTools Evidence**:
- **Components Tab**: Selecting `NewsletterForm` shows no state or hooks. The component is rendered once and never again, no matter how much you type.
- **Profiler Tab**: If you record a profiling session while typing, you will see zero re-renders for the `NewsletterForm` component.

**Let's parse this evidence**:

1.  **What the user experiences**: The UI is broken. It doesn't reflect the data it's supposed to.
    -   **Expected**: As I type "test@example.com", the text below the form should update to show "Current email: test@example.com".
    -   **Actual**: The text remains "Current email: ".

2.  **What the console reveals**: The JavaScript logic is working correctly. The `email` variable holds the right value at every keystroke. This tells us the problem isn't with our event handler logic itself.

3.  **What DevTools shows**: The key piece of evidence is that **React is not re-rendering the component**.

4.  **Root cause identified**: React has no mechanism to "watch" for changes in regular JavaScript variables. It only re-renders a component in response to specific triggers: a change in its props or a change in its state.

5.  **Why the current approach can't solve this**: A component's render is like a snapshot in time. Our `email` variable is created from scratch (`let email = ''`) every single time the component function runs. Even though we update it, that updated value is lost the next time the component renders (if it ever does). We need a way to preserve this value *between* renders.

6.  **What we need**: We need a way to tell React: "Here is a piece of data that I want you to remember. When this data changes, please re-render this component so the UI is up-to-date." This is precisely what the `useState` hook is for.

### Iteration 2: The Right Way - Introducing `useState`

Let's fix our component using `useState`.

First, you must import it from React. Then, you call it at the top level of your component.

`useState` returns an array with exactly two items:
1.  The **current state value**.
2.  A **setter function** to update that value.

We use array destructuring to give them names. The convention is `[thing, setThing]`.

**Before** (Iteration 1):

```tsx
// src/components/NewsletterForm.tsx (Attempt 1)
import React from 'react'; // Assuming React is imported

function NewsletterForm() {
  let email = ''; // Regular variable

  const handleChange = (event) => {
    email = event.target.value;
    console.log('Variable updated:', email);
  };

  // ... rest of the component
  return (
    // ...
    <p style={{ marginTop: '16px' }}>Current email: {email}</p>
    // ...
  );
}
```

**After** (Iteration 2 - The Fix):

```tsx
// src/components/NewsletterForm.tsx (Version 2 - With useState)
import React, { useState } from 'react'; // 1. Import useState

function NewsletterForm() {
  // 2. Call useState to declare a state variable
  const [email, setEmail] = useState(''); // `email` is the state, `setEmail` is the updater

  const handleChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    // 3. Use the setter function to update state
    setEmail(event.target.value); 
  };

  return (
    <div style={{ border: '1px solid #ccc', padding: '16px', borderRadius: '8px', maxWidth: '400px' }}>
      <h2>Subscribe to our Newsletter</h2>
      <p>Enter your email to get the latest updates.</p>
      <input
        type="email"
        placeholder="you@example.com"
        onChange={handleChange}
        value={email} // 4. Bind the input's value to our state variable
        style={{ width: 'calc(100% - 10px)', padding: '8px', marginBottom: '8px' }}
      />
      <button style={{ width: '100%', padding: '10px', background: '#0070f3', color: 'white', border: 'none', borderRadius: '4px' }}>
        Subscribe
      </button>
      <p style={{ marginTop: '16px' }}>Current email: {email}</p>
    </div>
  );
}

export default NewsletterForm;
```

### Verification and Analysis

Now, when you run this new version:

-   **Browser Behavior**: As you type in the input field, the text "Current email:" updates with every keystroke. The UI is perfectly in sync with your input.
-   **React DevTools**:
    -   **Components Tab**: Selecting `NewsletterForm` now shows a `State` hook. You can see the value change in real-time as you type.
    -   **Profiler Tab**: Recording a session while typing shows that `NewsletterForm` re-renders on every keystroke, which is exactly what we want. The reason for the render is listed as "Hook 1 changed".

**What happened?**

1.  We called `useState('')`, telling React to create a piece of state for this component, initialized to an empty string.
2.  React gives us back `email` (the current value, `''`) and `setEmail` (the function to change it).
3.  When the user types, `handleChange` is called.
4.  Inside `handleChange`, we call `setEmail(event.target.value)`.
5.  This tells React: "The state for this component has changed. The new value is what the user just typed."
6.  React schedules a re-render of `NewsletterForm`.
7.  When `NewsletterForm` re-renders, it calls `useState('')` again. This time, React doesn't give back the initial value; it gives back the *latest* value we set ("test@example.com").
8.  The JSX is returned with the new `email` value, and the browser displays the updated UI.

This is the declarative nature of React. We don't manually change the DOM. We **declare what the UI should look like based on the current state**, and React handles the rest.

## Event Handling in React

## Responding to User Actions

Our form now correctly tracks the user's input in its state. The next step is to make the "Subscribe" button do something. This requires handling user events.

React's event system is a wrapper around the browser's native event system. It provides a few key benefits:
-   **Cross-browser compatibility**: React normalizes events so they behave consistently across different browsers.
-   **CamelCase Naming**: Event names are written in camelCase, like `onClick` and `onChange`, instead of the lowercase HTML attributes `onclick` and `onchange`.
-   **Function References**: You pass a function reference (e.g., `{handleClick}`) to the event handler, not a string of code.

### Iteration 3: Handling a Button Click

Our component now has state, but the button is still inert. Let's add a function to handle the click event.

**Current state recap**: Our component uses `useState` to track the email input.
**Current limitation**: The "Subscribe" button does nothing when clicked.
**New scenario introduction**: What happens when a user clicks "Subscribe"? We want to simulate submitting their email address. For now, we'll just log it to the console.

Let's define a `handleSubmit` function and attach it to the button's `onClick` prop.

```tsx
// src/components/NewsletterForm.tsx (Version 3 - With Event Handling)
import React, { useState } from 'react';

function NewsletterForm() {
  const [email, setEmail] = useState('');

  const handleChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    setEmail(event.target.value);
  };

  // 1. Define the event handler function
  const handleSubmit = () => {
    // For now, we'll just show an alert and log the state
    alert(`Subscribing with email: ${email}`);
    console.log('Form submitted with email:', email);
  };

  return (
    <div style={{ border: '1px solid #ccc', padding: '16px', borderRadius: '8px', maxWidth: '400px' }}>
      <h2>Subscribe to our Newsletter</h2>
      <p>Enter your email to get the latest updates.</p>
      <input
        type="email"
        placeholder="you@example.com"
        onChange={handleChange}
        value={email}
        style={{ width: 'calc(100% - 10px)', padding: '8px', marginBottom: '8px' }}
      />
      {/* 2. Attach the handler to the onClick prop */}
      <button 
        onClick={handleSubmit} 
        style={{ width: '100%', padding: '10px', background: '#0070f3', color: 'white', border: 'none', borderRadius: '4px' }}
      >
        Subscribe
      </button>
      <p style={{ marginTop: '16px' }}>Current email: {email}</p>
    </div>
  );
}

export default NewsletterForm;
```

### Verification

Run this code, type an email into the input, and click the "Subscribe" button.

-   **Browser Behavior**: An alert box appears with the message "Subscribing with email: [your typed email]".
-   **Browser Console Output**:
    ```
    Form submitted with email: test@example.com
    ```

This confirms that our `handleSubmit` function was called and, crucially, that it had access to the latest `email` value from our component's state.

### Common Failure Mode: Calling the Function Instead of Passing It

A very common mistake for beginners is to accidentally *call* the function in the JSX.

**Symptom**: The alert box appears immediately when the component first renders, and then it doesn't work when you click the button.

**Code Causing the Failure**:
```tsx
// WRONG! Do not do this.
<button onClick={handleSubmit()}> 
  Subscribe
</button>
```

**Diagnostic Analysis**:

-   **Browser Behavior**: The `alert` fires the moment the component loads, before you've even had a chance to click anything. Clicking the button afterwards does nothing.
-   **Console Pattern**: You might see a warning from React.
    ```
    Warning: Expected `onClick` listener to be a function, instead got a value of `undefined` type.
    ```
-   **Root Cause**: When the JSX is being evaluated, `handleSubmit()` is executed immediately. The `onClick` prop is then assigned the *return value* of `handleSubmit`. Since our function doesn't return anything, `onClick` gets `undefined`.
-   **Solution**: Always pass a function reference, not the result of a function call.
    -   **Correct**: `onClick={handleSubmit}`
    -   **Incorrect**: `onClick={handleSubmit()}`

If you need to pass arguments, use an inline arrow function:
`onClick={() => someFunction(arg1)}`

## Controlled vs. Uncontrolled Components

## The Single Source of Truth

In React, there's an important concept for form elements like `<input>`, `<textarea>`, and `<select>`: the distinction between **controlled** and **uncontrolled** components.

-   A **Controlled Component** is an input element whose value is controlled by React state. The state is the "single source of truth." We've already built one! Our `NewsletterForm` is a controlled component because we set `value={email}` on the input.
-   An **Uncontrolled Component** is an input element where the DOM handles its own state internally. You might read its value when needed using a `ref`.

The idiomatic and recommended approach in React is to use controlled components. They make it easier to manage form data, implement instant validation, and conditionally disable/enable buttons.

### Iteration 4: The Dangers of a "Half-Controlled" Component

Let's demonstrate *why* controlled components are so important by breaking our current implementation slightly. What happens if we keep the `onChange` handler but remove the `value` prop from the input?

**Current state recap**: Our form uses `useState`, `onChange`, and `onClick`. The input's `value` is bound to the `email` state.
**Current limitation**: The importance of the `value` prop isn't obvious. It seems to work without it.
**New scenario introduction**: Let's add a "Reset" button that clears the form. This will expose the weakness of not having a fully controlled component.

**Before** (Fully Controlled):

```tsx
// src/components/NewsletterForm.tsx (Version 3)
// ...
const [email, setEmail] = useState('');
// ...
return (
  // ...
  <input
    onChange={handleChange}
    value={email} // ← The key part
  />
  // ...
);
```

**After** (The Broken Version):
We will remove the `value` prop and add a Reset button.

```tsx
// src/components/NewsletterForm.tsx (Attempt 4 - This will be buggy)
import React, { useState } from 'react';

function NewsletterForm() {
  const [email, setEmail] = useState('');

  const handleChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    setEmail(event.target.value);
  };

  const handleSubmit = () => {
    alert(`Subscribing with email: ${email}`);
  };

  const handleReset = () => {
    console.log('Resetting state...');
    setEmail(''); // Attempt to clear the input
  };

  return (
    <div style={{ border: '1px solid #ccc', padding: '16px', borderRadius: '8px', maxWidth: '400px' }}>
      <h2>Subscribe to our Newsletter</h2>
      <input
        type="email"
        placeholder="you@example.com"
        onChange={handleChange}
        // value={email} <-- We have removed this prop!
        style={{ width: 'calc(100% - 10px)', padding: '8px', marginBottom: '8px' }}
      />
      <div style={{ display: 'flex', gap: '8px' }}>
        <button onClick={handleSubmit} style={{ flex: 1, padding: '10px', background: '#0070f3', color: 'white', border: 'none', borderRadius: '4px' }}>
          Subscribe
        </button>
        <button onClick={handleReset} style={{ flex: 1, padding: '10px', background: '#666', color: 'white', border: 'none', borderRadius: '4px' }}>
          Reset
        </button>
      </div>
      <p style={{ marginTop: '16px' }}>Current email in state: {email}</p>
    </div>
  );
}

export default NewsletterForm;
```

### Diagnostic Analysis: Reading the Failure

1.  Type "test@example.com" into the input field. The text "Current email in state: test@example.com" appears correctly. So far, so good.
2.  Now, click the "Reset" button.

**Browser Behavior**:
The text below the form updates to "Current email in state: ", but the text "test@example.com" **remains visible inside the input box**. The UI is now out of sync with the state.

**Browser Console Output**:
```
Resetting state...
```
The `handleReset` function was called correctly.

**React DevTools Evidence**:
- **Components Tab**: After clicking "Reset", the `State` hook for `NewsletterForm` correctly shows an empty string (`''`). However, the DOM inspector shows the `<input>` element still has its `value` attribute set to "test@example.com".

**Let's parse this evidence**:

1.  **What the user experiences**: The reset button appears broken. It clears the state display but not the actual input field.
    -   **Expected**: Clicking "Reset" should clear both the state display and the text inside the input box.
    -   **Actual**: Only the state display is cleared.

2.  **What the console reveals**: Our event handler logic is correct.

3.  **What DevTools shows**: The React state and the DOM state have diverged. React thinks the value is `''`, but the browser's DOM thinks the value is `"test@example.com"`.

4.  **Root cause identified**: By removing the `value={email}` prop, we broke the connection from our React state *to* the input field. We told the DOM, "You are in charge of your own value." Our `onChange` handler still listens to the DOM and updates React state, but React has no way to command the DOM to change its value.

5.  **Why the current approach can't solve this**: Without the `value` prop, the data flow is one-way: `DOM -> React`. We need a two-way data flow: `DOM -> React` (via `onChange`) and `React -> DOM` (via `value`).

6.  **What we need**: We need to re-establish React as the single source of truth by adding back the `value` prop, making the component fully controlled.

### The Fix: Restoring Control

The solution is simple: add the `value={email}` prop back to the `<input>` element.

```tsx
// ...
<input
  type="email"
  placeholder="you@example.com"
  onChange={handleChange}
  value={email} // ← The fix!
  style={{ width: 'calc(100% - 10px)', padding: '8px', marginBottom: '8px' }}
/>
// ...
```

With this one line restored, the "Reset" button now works perfectly. When `handleReset` calls `setEmail('')`, React re-renders the component. During the render, it sees `<input value={''}>` and instructs the DOM to update the input field's value to an empty string. The UI and state are once again in perfect sync.

## Building Interactive UIs

## Synthesizing State and Events

We've learned how to manage data with `useState` and respond to user actions with event handlers. Now let's combine these concepts to build a more robust and user-friendly version of our `NewsletterForm`.

A real-world form needs to handle more than just success. It needs loading states to give feedback during network requests and error states to inform the user when something goes wrong. We can manage all of this with more state.

### Iteration 5: Adding Submission Status

Let's evolve our form to be more realistic. We'll add a new piece of state, `status`, which can be one of four values: `'idle'`, `'submitting'`, `'success'`, or `'error'`. The UI will change based on this status.

-   When `submitting`, the button will be disabled and show "Subscribing...".
-   On `success`, we'll show a success message.
-   On `error`, we'll show an error message.

We will also switch from a button `onClick` to a form `onSubmit` handler, which is the semantically correct way to handle form submissions. This also allows users to submit the form by pressing "Enter" in the input field.

```tsx
// src/components/NewsletterForm.tsx (Version 5 - Production-Ready)
import React, { useState } from 'react';

// Define a type for our status for better TypeScript support
type FormStatus = 'idle' | 'submitting' | 'success' | 'error';

function NewsletterForm() {
  const [email, setEmail] = useState('');
  const [status, setStatus] = useState<FormStatus>('idle');
  const [errorMessage, setErrorMessage] = useState<string | null>(null);

  const handleSubmit = async (event: React.FormEvent<HTMLFormElement>) => {
    event.preventDefault(); // Prevent the default browser page reload
    
    setStatus('submitting');
    setErrorMessage(null);

    // Simulate a network request
    try {
      await new Promise((resolve, reject) => {
        setTimeout(() => {
          if (email.includes('fail')) {
            reject(new Error('This email address is blocked.'));
          } else {
            resolve(true);
          }
        }, 2000); // 2-second delay
      });

      setStatus('success');
      setEmail('');
    } catch (e) {
      setStatus('error');
      setErrorMessage((e as Error).message);
    }
  };

  return (
    <form 
      onSubmit={handleSubmit}
      style={{ border: '1px solid #ccc', padding: '16px', borderRadius: '8px', maxWidth: '400px' }}
    >
      <h2>Subscribe to our Newsletter</h2>
      
      {status === 'success' ? (
        <p style={{ color: 'green' }}>Thank you for subscribing!</p>
      ) : (
        <p>Enter your email to get the latest updates.</p>
      )}

      <input
        type="email"
        placeholder="you@example.com"
        onChange={(e) => setEmail(e.target.value)}
        value={email}
        disabled={status === 'submitting'}
        required
        style={{ width: 'calc(100% - 10px)', padding: '8px', marginBottom: '8px' }}
      />
      <button 
        type="submit"
        disabled={status === 'submitting'}
        style={{ width: '100%', padding: '10px', background: '#0070f3', color: 'white', border: 'none', borderRadius: '4px', opacity: status === 'submitting' ? 0.7 : 1 }}
      >
        {status === 'submitting' ? 'Subscribing...' : 'Subscribe'}
      </button>

      {status === 'error' && (
        <p style={{ color: 'red', marginTop: '8px' }}>Error: {errorMessage}</p>
      )}
    </form>
  );
}

export default NewsletterForm;
```

### Analysis of the Final Version

This final component is a microcosm of a modern React application. Let's break down how it uses state and events together:

1.  **Multiple State Variables**: We use three `useState` hooks to manage different aspects of the component's memory: the user's input (`email`), the form's current state (`status`), and any potential error messages (`errorMessage`).
2.  **Conditional Rendering**: The UI changes dramatically based on the `status` state. We use ternary operators (`condition ? A : B`) and logical AND (`condition && A`) to show or hide different elements. This is a core pattern in React.
3.  **Form Submission Handling**: We use the `<form>` element's `onSubmit` event. This is better practice than a button's `onClick` for submissions. We call `event.preventDefault()` to stop the browser from doing a full-page refresh, allowing our React code to handle the submission logic.
4.  **Disabling Elements**: The `disabled` attribute on the `input` and `button` is bound to our state (`status === 'submitting'`). This provides crucial user feedback and prevents invalid actions, like submitting the form twice.
5.  **Asynchronous Logic**: The `handleSubmit` function is `async` to handle the simulated network request (`await new Promise(...)`). It correctly sets the status to `'submitting'` before the request and then to `'success'` or `'error'` after it completes.

This single component demonstrates the power of combining state and events. By declaring how the UI should look for each possible state, we can create complex, interactive, and user-friendly experiences.

### The Journey: From Problem to Solution

| Iteration | Failure Mode / Limitation                               | Technique Applied                               | Result                                                              |
| :-------- | :------------------------------------------------------ | :---------------------------------------------- | :------------------------------------------------------------------ |
| 0         | Static, non-interactive UI.                             | None (HTML/JSX only)                            | A "dead" component.                                                 |
| 1         | Using a regular variable doesn't trigger re-renders.    | `let email = ''`                                | UI does not update on user input.                                   |
| 2         | Component is not interactive.                           | `useState`                                      | State is tracked, and component re-renders on change.               |
| 3         | Button clicks do nothing.                               | `onClick` event handler                         | Button triggers a function that can access component state.         |
| 4         | UI and state can become out of sync (Reset button bug). | Controlled Component (`value` prop)             | React state becomes the single source of truth for the input value. |
| 5         | Lacks feedback for async operations (loading, errors).  | Multiple state variables, `onSubmit`, conditional rendering | A robust, production-ready form with clear user feedback.           |

### Lessons Learned

-   **State is Memory**: Use `useState` for any data that a component needs to remember and that can change over time.
-   **State Triggers Renders**: The only way to change what's on the screen in response to an event is to update state.
-   **Events Trigger State Changes**: User interactions (clicks, typing) are handled by event handlers, whose primary job is often to call a state setter function.
-   **Control Your Inputs**: Always use controlled components for forms by binding an input's `value` and `onChange` to React state. This makes your UI predictable and easy to manage.
-   **Describe Every State**: Think about all the possible states your component can be in (loading, success, error, empty, filled) and use state variables and conditional rendering to describe the UI for each one.
