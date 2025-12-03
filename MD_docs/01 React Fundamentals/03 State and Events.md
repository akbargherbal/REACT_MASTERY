# Chapter 3: State and Events

## The Failure: Why Variables Don't Work

## The Failure: Why Variables Don't Work

We're going to build a user profile dashboard that displays user information and allows editing. This will be our **anchor example** throughout this chapter—we'll refine it through multiple iterations as we discover React's state management patterns.

Let's start with what seems like the obvious approach: using regular JavaScript variables to track data that changes.

**Project Structure**:
```
src/
├── components/
│   └── UserProfile.tsx  ← Our reference implementation
└── App.tsx
```

Here's our first attempt at an editable user profile:

```tsx
// src/components/UserProfile.tsx
function UserProfile() {
  // Seems reasonable: store the user's name in a variable
  let userName = "Jane Smith";
  let userEmail = "jane@example.com";

  function handleNameChange() {
    userName = "Jane Doe"; // Update the variable
    console.log("Name updated to:", userName);
  }

  function handleEmailChange() {
    userEmail = "jane.doe@example.com"; // Update the variable
    console.log("Email updated to:", userEmail);
  }

  return (
    <div className="profile-card">
      <h2>User Profile</h2>
      <div className="profile-info">
        <p><strong>Name:</strong> {userName}</p>
        <p><strong>Email:</strong> {userEmail}</p>
      </div>
      <div className="profile-actions">
        <button onClick={handleNameChange}>
          Change Name
        </button>
        <button onClick={handleEmailChange}>
          Change Email
        </button>
      </div>
    </div>
  );
}

export default UserProfile;
```

```tsx
// src/App.tsx
import UserProfile from './components/UserProfile';

function App() {
  return (
    <div className="app">
      <UserProfile />
    </div>
  );
}

export default App;
```

This code compiles without errors. It runs without crashing. But when you click the buttons, something strange happens—or rather, nothing happens.

### Diagnostic Analysis: Reading React's Silence

**Browser Behavior**:
The UI shows "Jane Smith" and "jane@example.com". When you click "Change Name", nothing visible changes. The text stays exactly the same. No error appears. The button works (you can see it respond to clicks), but the displayed name doesn't update.

**Browser Console Output**:
```
Name updated to: Jane Doe
```

The console confirms our function ran. The variable was updated. But the UI didn't change.

**React DevTools Evidence**:
- Open React DevTools → Components tab
- Select the `UserProfile` component
- Props: (none)
- Hooks: (none)
- The component shows no state
- Click "Change Name" button
- Observe: Component does NOT re-render
- The render count stays at 1

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Clicking "Change Name" should update the displayed name to "Jane Doe"
   - Actual: Nothing changes in the UI, though the button responds to clicks

2. **What the console reveals**:
   - The function executes successfully
   - The variable is updated in memory
   - Key indicator: The log shows the new value, but the UI doesn't reflect it

3. **What DevTools shows**:
   - The component rendered exactly once (on mount)
   - No re-render occurred when the button was clicked
   - No hooks are present in the component

4. **Root cause identified**: React doesn't know the variable changed, so it doesn't re-render the component.

5. **Why the current approach can't solve this**: Regular JavaScript variables are invisible to React's rendering system. React only re-renders components when it's explicitly told something changed.

6. **What we need**: A way to tell React "this data changed, please re-render the component."

### The Fundamental Problem

React components are **functions that return UI**. When React calls your component function, it:

1. Executes the function
2. Gets the JSX you returned
3. Updates the DOM to match that JSX
4. **Stops**

React doesn't continuously watch your variables. It doesn't poll for changes. It renders once and waits for you to tell it to render again.

When you write `userName = "Jane Doe"`, you're updating a local variable inside a function. The next time React calls that function (if it ever does), `userName` will be back to `"Jane Smith"` because the function starts fresh each time.

**The variable changes, but React never re-renders, so the UI never updates.**

This is not a bug in React—it's a fundamental design decision. React gives you explicit control over when components re-render. This control is what makes React predictable and performant.

## useState: Local Component State

## useState: Local Component State

React provides `useState` to solve exactly this problem: tracking data that changes over time and triggering re-renders when it does.

### Iteration 1: Making the Profile Interactive

Let's fix our component using `useState`:

```tsx
// src/components/UserProfile.tsx
import { useState } from 'react';

function UserProfile() {
  // useState returns [currentValue, functionToUpdateIt]
  const [userName, setUserName] = useState("Jane Smith");
  const [userEmail, setUserEmail] = useState("jane@example.com");

  function handleNameChange() {
    setUserName("Jane Doe"); // Tell React to update and re-render
    console.log("Name change requested");
  }

  function handleEmailChange() {
    setUserEmail("jane.doe@example.com");
    console.log("Email change requested");
  }

  console.log("Component rendering with:", userName, userEmail);

  return (
    <div className="profile-card">
      <h2>User Profile</h2>
      <div className="profile-info">
        <p><strong>Name:</strong> {userName}</p>
        <p><strong>Email:</strong> {userEmail}</p>
      </div>
      <div className="profile-actions">
        <button onClick={handleNameChange}>
          Change Name
        </button>
        <button onClick={handleEmailChange}>
          Change Email
        </button>
      </div>
    </div>
  );
}

export default UserProfile;
```

Now when you click "Change Name":

**Browser Console Output**:
```
Component rendering with: Jane Smith jane@example.com
Name change requested
Component rendering with: Jane Doe jane@example.com
```

**Browser Behavior**:
The name updates immediately from "Jane Smith" to "Jane Doe". The UI reflects the change.

**React DevTools Evidence**:
- Components tab → `UserProfile` selected
- Hooks: `State: "Jane Smith"`, `State: "jane@example.com"`
- Click "Change Name"
- Observe: Component re-renders (render count increases)
- First hook now shows: `State: "Jane Doe"`
- The component tree briefly highlights, indicating a re-render

**Expected vs. Actual improvement**:
- Before: Variable changed, but UI stayed frozen at initial value
- After: UI updates immediately when button is clicked
- Evidence: Console shows two render logs—initial render and post-update render

### How useState Works

Let's break down what's happening:

```typescript
const [userName, setUserName] = useState("Jane Smith");
```

This line does three things:

1. **On first render**: Creates a state variable initialized to `"Jane Smith"`
2. **Returns the current value**: `userName` is `"Jane Smith"`
3. **Returns an updater function**: `setUserName` is a function that:
   - Updates the state value
   - Schedules a re-render of the component
   - Preserves the new value between renders

When you call `setUserName("Jane Doe")`:

1. React updates its internal state storage
2. React schedules a re-render of `UserProfile`
3. React calls your component function again
4. This time, `useState("Jane Smith")` returns `"Jane Doe"` (the updated value)
5. Your component returns new JSX with the updated name
6. React updates the DOM to match

**The key insight**: `useState` gives React a handle to track your data. When you call the setter function, you're not just updating a variable—you're telling React "this component needs to re-render."

### The Anatomy of useState

```typescript
// The pattern:
const [value, setValue] = useState(initialValue);

// Breaking it down:
// - useState is a function that takes an initial value
// - It returns an array with exactly two elements
// - We use array destructuring to name them

// Element 0: The current state value
// Element 1: A function to update that value

// Naming convention:
// - State variable: descriptive name (userName, isLoading, count)
// - Setter function: "set" + capitalized variable name (setUserName, setIsLoading, setCount)
```

### Multiple State Variables

Notice we used `useState` twice:

```typescript
const [userName, setUserName] = useState("Jane Smith");
const [userEmail, setUserEmail] = useState("jane@example.com");
```

Each `useState` call creates an independent piece of state. Updating one doesn't affect the other. This is different from class components (which you might see in older React code) where all state lived in a single object.

**Why separate state variables?**

- **Clarity**: Each piece of state has a clear purpose
- **Independence**: Updating name doesn't require knowing about email
- **Simplicity**: No need to merge objects or worry about overwriting other state

### State Updates Are Asynchronous

Here's a common mistake:

```tsx
function UserProfile() {
  const [userName, setUserName] = useState("Jane Smith");

  function handleNameChange() {
    setUserName("Jane Doe");
    console.log("Name is now:", userName); // ⚠️ Still "Jane Smith"!
  }

  return (
    <div>
      <p>{userName}</p>
      <button onClick={handleNameChange}>Change Name</button>
    </div>
  );
}
```

**Browser Console Output**:
```
Name is now: Jane Smith
```

The console shows the *old* value, not the new one. Why?

**State updates are asynchronous**. When you call `setUserName("Jane Doe")`, React:

1. Schedules the update
2. Continues executing your function
3. Later, re-renders with the new value

The `userName` variable in your current function execution still holds the old value. The new value will be available in the *next* render.

If you need to do something with the new value, do it in the next render:

```tsx
function UserProfile() {
  const [userName, setUserName] = useState("Jane Smith");

  function handleNameChange() {
    setUserName("Jane Doe");
    // Don't try to use the new value here
  }

  // The new value is available here, in the render
  console.log("Current name:", userName);

  return (
    <div>
      <p>{userName}</p>
      <button onClick={handleNameChange}>Change Name</button>
    </div>
  );
}
```

### Limitation Preview

Our profile now updates when buttons are clicked, but it's not very flexible. The new values are hardcoded. What if we want users to type their own names? We need to connect state to input fields—that's where **controlled components** come in (Section 3.3).

## Event Handling in React

## Event Handling in React

Before we make our profile truly editable with input fields, let's understand how React handles events.

### Iteration 2: Adding Real Input Fields

Let's replace our hardcoded buttons with actual text inputs:

```tsx
// src/components/UserProfile.tsx
import { useState } from 'react';

function UserProfile() {
  const [userName, setUserName] = useState("Jane Smith");
  const [userEmail, setUserEmail] = useState("jane@example.com");

  // Handle input changes
  function handleNameChange(event) {
    console.log("Input event:", event);
    console.log("New value:", event.target.value);
    setUserName(event.target.value);
  }

  function handleEmailChange(event) {
    setUserEmail(event.target.value);
  }

  return (
    <div className="profile-card">
      <h2>User Profile</h2>
      <div className="profile-form">
        <div className="form-field">
          <label htmlFor="name">Name:</label>
          <input
            id="name"
            type="text"
            value={userName}
            onChange={handleNameChange}
          />
        </div>
        <div className="form-field">
          <label htmlFor="email">Email:</label>
          <input
            id="email"
            type="email"
            value={userEmail}
            onChange={handleEmailChange}
          />
        </div>
      </div>
      <div className="profile-display">
        <p><strong>Display Name:</strong> {userName}</p>
        <p><strong>Display Email:</strong> {userEmail}</p>
      </div>
    </div>
  );
}

export default UserProfile;
```

**Browser Behavior**:
Type in the name input field. Each keystroke immediately updates the "Display Name" below. The input field and the display stay perfectly synchronized.

**Browser Console Output** (when typing "Jane Doe"):
```
Input event: SyntheticBaseEvent {_reactName: 'onChange', ...}
New value: Jane D
Input event: SyntheticBaseEvent {_reactName: 'onChange', ...}
New value: Jane Do
Input event: SyntheticBaseEvent {_reactName: 'onChange', ...}
New value: Jane Doe
```

### React Events vs. Native DOM Events

React wraps native browser events in a `SyntheticEvent` object. This provides:

1. **Cross-browser consistency**: The same event API works in all browsers
2. **Performance**: React reuses event objects (they're pooled)
3. **React integration**: Events work seamlessly with React's rendering

The event object you receive in your handler has the same interface as native DOM events:

- `event.target` - The element that triggered the event
- `event.target.value` - The current value of an input
- `event.preventDefault()` - Prevent default browser behavior
- `event.stopPropagation()` - Stop event bubbling

### Event Handler Patterns

React supports all standard DOM events with camelCase names:

```tsx
// Click events
<button onClick={handleClick}>Click Me</button>

// Form events
<input onChange={handleChange} />
<form onSubmit={handleSubmit} />

// Mouse events
<div onMouseEnter={handleMouseEnter} onMouseLeave={handleMouseLeave} />

// Keyboard events
<input onKeyDown={handleKeyDown} onKeyUp={handleKeyUp} />

// Focus events
<input onFocus={handleFocus} onBlur={handleBlur} />
```

### Inline Event Handlers

You can define event handlers inline using arrow functions:

```tsx
function UserProfile() {
  const [userName, setUserName] = useState("Jane Smith");

  return (
    <input
      value={userName}
      onChange={(event) => setUserName(event.target.value)}
    />
  );
}
```

This is perfectly fine for simple handlers. For more complex logic, extract to a named function for readability.

### Passing Arguments to Event Handlers

Sometimes you need to pass additional data to your handler:

```tsx
function UserProfile() {
  const [activeTab, setActiveTab] = useState("profile");

  // Option 1: Inline arrow function
  return (
    <div>
      <button onClick={() => setActiveTab("profile")}>
        Profile
      </button>
      <button onClick={() => setActiveTab("settings")}>
        Settings
      </button>
      <button onClick={() => setActiveTab("activity")}>
        Activity
      </button>
    </div>
  );
}
```

```tsx
function UserProfile() {
  const [activeTab, setActiveTab] = useState("profile");

  // Option 2: Function that returns a function
  function handleTabClick(tabName: string) {
    return () => {
      setActiveTab(tabName);
    };
  }

  return (
    <div>
      <button onClick={handleTabClick("profile")}>Profile</button>
      <button onClick={handleTabClick("settings")}>Settings</button>
      <button onClick={handleTabClick("activity")}>Activity</button>
    </div>
  );
}
```

### Common Event Handling Mistakes

#### Mistake 1: Calling the Function Instead of Passing It

```tsx
// ❌ Wrong: This calls handleClick immediately during render
<button onClick={handleClick()}>Click Me</button>

// ✅ Correct: This passes the function to be called later
<button onClick={handleClick}>Click Me</button>

// ✅ Also correct: Arrow function that calls it when clicked
<button onClick={() => handleClick()}>Click Me</button>
```

**The Failure**:

When you write `onClick={handleClick()}`, the function executes during render, not when clicked. If `handleClick` updates state, this causes an infinite loop:

1. Component renders
2. `handleClick()` executes, updates state
3. State update triggers re-render
4. Component renders again
5. `handleClick()` executes again...
6. Infinite loop

**Browser Console Output**:
```
Warning: Maximum update depth exceeded. This can happen when a component 
calls setState inside useEffect, but useEffect either doesn't have a 
dependency array, or one of the dependencies changes on every render.
```

**React DevTools Evidence**:
- Profiler shows thousands of renders in seconds
- Component render count increases rapidly
- Browser becomes unresponsive

#### Mistake 2: Forgetting event.preventDefault()

```tsx
// ❌ Form submits and page reloads
function handleSubmit(event) {
  setUserName(event.target.name.value);
  // Page reloads here, losing all state
}

// ✅ Prevent default form submission
function handleSubmit(event) {
  event.preventDefault(); // Stop the page reload
  setUserName(event.target.name.value);
}

return (
  <form onSubmit={handleSubmit}>
    <input name="name" />
    <button type="submit">Save</button>
  </form>
);
```

### Limitation Preview

Our inputs now work, but there's a subtle issue. Try this: remove the `value={userName}` prop from the input. The input still works, but now the "Display Name" and the input can get out of sync. This is the difference between **controlled** and **uncontrolled** components—our next topic.

## Controlled vs. Uncontrolled Components

## Controlled vs. Uncontrolled Components

### The Failure: Uncontrolled Inputs

Let's see what happens when we remove the `value` prop from our input:

```tsx
// src/components/UserProfile.tsx
import { useState } from 'react';

function UserProfile() {
  const [userName, setUserName] = useState("Jane Smith");

  function handleNameChange(event) {
    setUserName(event.target.value);
  }

  function resetName() {
    setUserName("Jane Smith"); // Try to reset to original
  }

  return (
    <div className="profile-card">
      <h2>User Profile</h2>
      <div className="form-field">
        <label htmlFor="name">Name:</label>
        <input
          id="name"
          type="text"
          // No value prop - uncontrolled
          onChange={handleNameChange}
        />
      </div>
      <div className="profile-display">
        <p><strong>Display Name:</strong> {userName}</p>
      </div>
      <button onClick={resetName}>Reset Name</button>
    </div>
  );
}

export default UserProfile;
```

**Browser Behavior**:

1. Type "John Doe" in the input
2. Display Name updates to "John Doe" ✓
3. Click "Reset Name" button
4. Display Name changes back to "Jane Smith" ✓
5. **But the input still shows "John Doe"** ✗

The input and the state are now out of sync.

### Diagnostic Analysis: The Uncontrolled Input Problem

**Browser Console Output**:
```
(No errors - this is valid React code)
```

**React DevTools Evidence**:
- Components tab → `UserProfile` selected
- Hooks: `State: "Jane Smith"` (after reset)
- The state is correct
- But the input's DOM value is still "John Doe"

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Clicking "Reset Name" should clear the input back to "Jane Smith"
   - Actual: The display updates, but the input keeps showing "John Doe"

2. **What DevTools shows**:
   - React's state is correct ("Jane Smith")
   - The component re-rendered
   - But the input's value didn't change

3. **Root cause identified**: The input is **uncontrolled**—React doesn't control its value. The input manages its own internal state (the DOM's value), separate from React's state.

4. **Why the current approach can't solve this**: Without the `value` prop, React can't tell the input what to display. The input's value is controlled by the DOM, not by React.

5. **What we need**: Make the input **controlled** by giving React full authority over its value.

### Controlled Components: React as the Single Source of Truth

A **controlled component** is an input whose value is controlled by React state:

```tsx
// src/components/UserProfile.tsx
import { useState } from 'react';

function UserProfile() {
  const [userName, setUserName] = useState("Jane Smith");

  function handleNameChange(event) {
    setUserName(event.target.value);
  }

  function resetName() {
    setUserName("Jane Smith");
  }

  return (
    <div className="profile-card">
      <h2>User Profile</h2>
      <div className="form-field">
        <label htmlFor="name">Name:</label>
        <input
          id="name"
          type="text"
          value={userName}  // ← Controlled: React controls the value
          onChange={handleNameChange}
        />
      </div>
      <div className="profile-display">
        <p><strong>Display Name:</strong> {userName}</p>
      </div>
      <button onClick={resetName}>Reset Name</button>
    </div>
  );
}

export default UserProfile;
```

**Browser Behavior**:

1. Type "John Doe" in the input
2. Display Name updates to "John Doe" ✓
3. Click "Reset Name" button
4. Display Name changes back to "Jane Smith" ✓
5. **Input also changes back to "Jane Smith"** ✓

Now the input and state stay perfectly synchronized.

**Expected vs. Actual improvement**:
- Before: Input had its own state, could diverge from React state
- After: Input value is always exactly what React state says it should be
- Evidence: Reset button now clears the input field

### How Controlled Components Work

With a controlled component, the data flow is:

1. User types in input
2. `onChange` event fires
3. Event handler calls `setUserName(newValue)`
4. React updates state
5. React re-renders component
6. Input receives new `value` prop
7. Input displays the value from React state

**React state is the single source of truth.** The input always displays what React tells it to display.

### Iteration 3: A Complete Editable Profile Form

Let's build a more realistic profile form with multiple controlled inputs:

```tsx
// src/components/UserProfile.tsx
import { useState } from 'react';

function UserProfile() {
  const [userName, setUserName] = useState("Jane Smith");
  const [userEmail, setUserEmail] = useState("jane@example.com");
  const [userBio, setUserBio] = useState("Software developer");
  const [isPublic, setIsPublic] = useState(true);

  function handleSubmit(event) {
    event.preventDefault();
    console.log("Profile saved:", {
      name: userName,
      email: userEmail,
      bio: userBio,
      isPublic: isPublic
    });
    alert("Profile saved!");
  }

  function handleReset() {
    setUserName("Jane Smith");
    setUserEmail("jane@example.com");
    setUserBio("Software developer");
    setIsPublic(true);
  }

  return (
    <div className="profile-card">
      <h2>Edit Profile</h2>
      <form onSubmit={handleSubmit}>
        <div className="form-field">
          <label htmlFor="name">Name:</label>
          <input
            id="name"
            type="text"
            value={userName}
            onChange={(e) => setUserName(e.target.value)}
          />
        </div>

        <div className="form-field">
          <label htmlFor="email">Email:</label>
          <input
            id="email"
            type="email"
            value={userEmail}
            onChange={(e) => setUserEmail(e.target.value)}
          />
        </div>

        <div className="form-field">
          <label htmlFor="bio">Bio:</label>
          <textarea
            id="bio"
            value={userBio}
            onChange={(e) => setUserBio(e.target.value)}
            rows={4}
          />
        </div>

        <div className="form-field">
          <label>
            <input
              type="checkbox"
              checked={isPublic}
              onChange={(e) => setIsPublic(e.target.checked)}
            />
            Make profile public
          </label>
        </div>

        <div className="form-actions">
          <button type="submit">Save Profile</button>
          <button type="button" onClick={handleReset}>
            Reset
          </button>
        </div>
      </form>

      <div className="profile-preview">
        <h3>Preview</h3>
        <p><strong>Name:</strong> {userName}</p>
        <p><strong>Email:</strong> {userEmail}</p>
        <p><strong>Bio:</strong> {userBio}</p>
        <p><strong>Visibility:</strong> {isPublic ? "Public" : "Private"}</p>
      </div>
    </div>
  );
}

export default UserProfile;
```

**Browser Behavior**:
- All inputs update the preview in real-time
- Reset button clears all fields back to defaults
- Submit button logs the form data and shows an alert
- Checkbox toggles between "Public" and "Private"

**Browser Console Output** (after editing and submitting):
```
Profile saved: {
  name: "John Doe",
  email: "john@example.com",
  bio: "Full-stack developer with 5 years experience",
  isPublic: false
}
```

### Controlled Component Patterns

#### Text Inputs

```tsx
const [value, setValue] = useState("");

<input
  type="text"
  value={value}
  onChange={(e) => setValue(e.target.value)}
/>
```

#### Textareas

```tsx
const [text, setText] = useState("");

<textarea
  value={text}
  onChange={(e) => setText(e.target.value)}
/>
```

Note: In React, `<textarea>` uses `value` prop, not children. This is different from HTML.

#### Checkboxes

```tsx
const [isChecked, setIsChecked] = useState(false);

<input
  type="checkbox"
  checked={isChecked}
  onChange={(e) => setIsChecked(e.target.checked)}
/>
```

Note: Checkboxes use `checked` prop and `e.target.checked`, not `value`.

#### Radio Buttons

```tsx
const [selectedOption, setSelectedOption] = useState("option1");

<div>
  <label>
    <input
      type="radio"
      value="option1"
      checked={selectedOption === "option1"}
      onChange={(e) => setSelectedOption(e.target.value)}
    />
    Option 1
  </label>
  <label>
    <input
      type="radio"
      value="option2"
      checked={selectedOption === "option2"}
      onChange={(e) => setSelectedOption(e.target.value)}
    />
    Option 2
  </label>
</div>
```

#### Select Dropdowns

```tsx
const [selected, setSelected] = useState("apple");

<select
  value={selected}
  onChange={(e) => setSelected(e.target.value)}
>
  <option value="apple">Apple</option>
  <option value="banana">Banana</option>
  <option value="orange">Orange</option>
</select>
```

### When to Use Uncontrolled Components

Controlled components are the React way, but uncontrolled components have their place:

**Use uncontrolled when**:
- You're integrating with non-React code
- You need to access the DOM directly (e.g., file inputs)
- The form is very simple and you don't need real-time validation

**File inputs are always uncontrolled** (you can't set their value programmatically for security reasons):

```tsx
function FileUpload() {
  function handleSubmit(event) {
    event.preventDefault();
    const file = event.target.file.files[0];
    console.log("Selected file:", file);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input type="file" name="file" />
      <button type="submit">Upload</button>
    </form>
  );
}
```

### Common Failure Modes and Their Signatures

#### Symptom: Input doesn't update when typing

**Browser behavior**:
You type in the input, but nothing appears. The cursor moves, but no text shows.

**Console pattern**:
```
Warning: You provided a `value` prop to a form field without an `onChange` handler.
This will render a read-only field.
```

**Root cause**: Input has `value` prop but no `onChange` handler. React makes it read-only.

**Solution**: Add `onChange` handler that updates state.

#### Symptom: Input shows "undefined" or "null"

**Browser behavior**:
Input displays the text "undefined" or "null" instead of being empty.

**Root cause**: State is `undefined` or `null`, but `value` prop expects a string.

**Solution**: Initialize state with empty string:

```tsx
// ❌ Wrong
const [name, setName] = useState();

// ✅ Correct
const [name, setName] = useState("");
```

#### Symptom: Checkbox doesn't toggle

**Browser behavior**:
Clicking checkbox doesn't change its state.

**Root cause**: Using `value` instead of `checked` prop, or `e.target.value` instead of `e.target.checked`.

**Solution**:

```tsx
// ❌ Wrong
<input
  type="checkbox"
  value={isChecked}
  onChange={(e) => setIsChecked(e.target.value)}
/>

// ✅ Correct
<input
  type="checkbox"
  checked={isChecked}
  onChange={(e) => setIsChecked(e.target.checked)}
/>
```

### Limitation Preview

Our form works, but managing multiple pieces of state with separate `useState` calls is getting verbose. What if we have 10 fields? 20? In Chapter 6, we'll learn about React Hook Form, which handles complex forms more elegantly. But first, we need to understand the patterns we're building on.

## Building Interactive UIs

## Building Interactive UIs

Now that we understand state and events, let's build something more interactive: a user dashboard with tabs, notifications, and dynamic content.

### Iteration 4: Multi-Tab Dashboard

Let's expand our profile into a full dashboard with multiple views:

```tsx
// src/components/UserDashboard.tsx
import { useState } from 'react';

function UserDashboard() {
  const [activeTab, setActiveTab] = useState("profile");
  const [userName, setUserName] = useState("Jane Smith");
  const [userEmail, setUserEmail] = useState("jane@example.com");
  const [notifications, setNotifications] = useState([
    { id: 1, message: "Welcome to your dashboard!", read: false },
    { id: 2, message: "Your profile is 80% complete", read: false },
    { id: 3, message: "New message from support", read: true }
  ]);

  function markAsRead(notificationId) {
    setNotifications(notifications.map(notif =>
      notif.id === notificationId
        ? { ...notif, read: true }
        : notif
    ));
  }

  function clearAllNotifications() {
    setNotifications([]);
  }

  return (
    <div className="dashboard">
      <header className="dashboard-header">
        <h1>User Dashboard</h1>
        <div className="user-info">
          <span>{userName}</span>
          <span className="notification-badge">
            {notifications.filter(n => !n.read).length}
          </span>
        </div>
      </header>

      <nav className="dashboard-tabs">
        <button
          className={activeTab === "profile" ? "active" : ""}
          onClick={() => setActiveTab("profile")}
        >
          Profile
        </button>
        <button
          className={activeTab === "notifications" ? "active" : ""}
          onClick={() => setActiveTab("notifications")}
        >
          Notifications
        </button>
        <button
          className={activeTab === "settings" ? "active" : ""}
          onClick={() => setActiveTab("settings")}
        >
          Settings
        </button>
      </nav>

      <main className="dashboard-content">
        {activeTab === "profile" && (
          <div className="profile-tab">
            <h2>Profile Information</h2>
            <div className="form-field">
              <label htmlFor="name">Name:</label>
              <input
                id="name"
                type="text"
                value={userName}
                onChange={(e) => setUserName(e.target.value)}
              />
            </div>
            <div className="form-field">
              <label htmlFor="email">Email:</label>
              <input
                id="email"
                type="email"
                value={userEmail}
                onChange={(e) => setUserEmail(e.target.value)}
              />
            </div>
          </div>
        )}

        {activeTab === "notifications" && (
          <div className="notifications-tab">
            <div className="notifications-header">
              <h2>Notifications</h2>
              <button onClick={clearAllNotifications}>
                Clear All
              </button>
            </div>
            {notifications.length === 0 ? (
              <p className="empty-state">No notifications</p>
            ) : (
              <ul className="notifications-list">
                {notifications.map(notif => (
                  <li
                    key={notif.id}
                    className={notif.read ? "read" : "unread"}
                  >
                    <span>{notif.message}</span>
                    {!notif.read && (
                      <button onClick={() => markAsRead(notif.id)}>
                        Mark as Read
                      </button>
                    )}
                  </li>
                ))}
              </ul>
            )}
          </div>
        )}

        {activeTab === "settings" && (
          <div className="settings-tab">
            <h2>Settings</h2>
            <p>Settings panel coming soon...</p>
          </div>
        )}
      </main>
    </div>
  );
}

export default UserDashboard;
```

**Browser Behavior**:
- Three tabs: Profile, Notifications, Settings
- Clicking tabs switches the content below
- Active tab is highlighted
- Notification badge shows unread count
- Marking notifications as read updates the badge
- Clear All removes all notifications and shows "No notifications"

**Browser Console Output** (when marking notification as read):
```
(No logs - but React DevTools shows state update)
```

**React DevTools Evidence**:
- Components tab → `UserDashboard` selected
- Hooks show multiple state values:
  - `State: "profile"` (activeTab)
  - `State: "Jane Smith"` (userName)
  - `State: "jane@example.com"` (userEmail)
  - `State: [Array(3)]` (notifications)
- Click "Mark as Read" on first notification
- Observe: Only the notifications state updates
- Component re-renders once
- The notification's `read` property changes from `false` to `true`

### State Management Patterns

#### Conditional Rendering Based on State

We're using state to control which tab content is visible:

```tsx
{activeTab === "profile" && (
  <div className="profile-tab">
    {/* Profile content */}
  </div>
)}

{activeTab === "notifications" && (
  <div className="notifications-tab">
    {/* Notifications content */}
  </div>
)}
```

This is a common pattern: use state to determine what to render. The `&&` operator short-circuits—if the left side is false, the right side never evaluates.

#### Updating Complex State (Arrays and Objects)

Notice how we update the notifications array:

```tsx
function markAsRead(notificationId) {
  setNotifications(notifications.map(notif =>
    notif.id === notificationId
      ? { ...notif, read: true }  // Create new object with updated property
      : notif                      // Keep existing object
  ));
}
```

**Key principle**: Never mutate state directly. Always create new arrays/objects.

**Why?** React compares the old state to the new state to decide if it needs to re-render. If you mutate the existing array, React sees the same array reference and might not re-render.

```tsx
// ❌ Wrong: Mutates existing array
function markAsRead(notificationId) {
  const notif = notifications.find(n => n.id === notificationId);
  notif.read = true; // Mutation!
  setNotifications(notifications); // Same array reference
}

// ✅ Correct: Creates new array
function markAsRead(notificationId) {
  setNotifications(notifications.map(notif =>
    notif.id === notificationId
      ? { ...notif, read: true }
      : notif
  ));
}
```

#### Derived State

The notification badge count is **derived state**—it's calculated from existing state, not stored separately:

```tsx
<span className="notification-badge">
  {notifications.filter(n => !n.read).length}
</span>
```

**Don't store derived state in useState**:

```tsx
// ❌ Wrong: Storing derived state
const [notifications, setNotifications] = useState([...]);
const [unreadCount, setUnreadCount] = useState(0);

// Now you have to keep them in sync manually
function markAsRead(id) {
  setNotifications(/* ... */);
  setUnreadCount(unreadCount - 1); // Easy to forget or get wrong
}

// ✅ Correct: Calculate derived state
const [notifications, setNotifications] = useState([...]);
const unreadCount = notifications.filter(n => !n.read).length;
```

**Why?** Derived state can't get out of sync if you calculate it on every render. It's always correct.

### Iteration 5: Adding Loading and Error States

Real applications need to handle loading and error states. Let's add them:

```tsx
// src/components/UserDashboard.tsx
import { useState } from 'react';

function UserDashboard() {
  const [activeTab, setActiveTab] = useState("profile");
  const [userName, setUserName] = useState("Jane Smith");
  const [userEmail, setUserEmail] = useState("jane@example.com");
  const [notifications, setNotifications] = useState([]);
  const [isLoadingNotifications, setIsLoadingNotifications] = useState(false);
  const [notificationError, setNotificationError] = useState(null);

  function loadNotifications() {
    setIsLoadingNotifications(true);
    setNotificationError(null);

    // Simulate API call
    setTimeout(() => {
      // Simulate random success/failure
      if (Math.random() > 0.3) {
        setNotifications([
          { id: 1, message: "Welcome to your dashboard!", read: false },
          { id: 2, message: "Your profile is 80% complete", read: false },
          { id: 3, message: "New message from support", read: true }
        ]);
        setIsLoadingNotifications(false);
      } else {
        setNotificationError("Failed to load notifications");
        setIsLoadingNotifications(false);
      }
    }, 1500);
  }

  function markAsRead(notificationId) {
    setNotifications(notifications.map(notif =>
      notif.id === notificationId
        ? { ...notif, read: true }
        : notif
    ));
  }

  return (
    <div className="dashboard">
      <header className="dashboard-header">
        <h1>User Dashboard</h1>
        <div className="user-info">
          <span>{userName}</span>
          {notifications.length > 0 && (
            <span className="notification-badge">
              {notifications.filter(n => !n.read).length}
            </span>
          )}
        </div>
      </header>

      <nav className="dashboard-tabs">
        <button
          className={activeTab === "profile" ? "active" : ""}
          onClick={() => setActiveTab("profile")}
        >
          Profile
        </button>
        <button
          className={activeTab === "notifications" ? "active" : ""}
          onClick={() => setActiveTab("notifications")}
        >
          Notifications
        </button>
      </nav>

      <main className="dashboard-content">
        {activeTab === "profile" && (
          <div className="profile-tab">
            <h2>Profile Information</h2>
            <div className="form-field">
              <label htmlFor="name">Name:</label>
              <input
                id="name"
                type="text"
                value={userName}
                onChange={(e) => setUserName(e.target.value)}
              />
            </div>
            <div className="form-field">
              <label htmlFor="email">Email:</label>
              <input
                id="email"
                type="email"
                value={userEmail}
                onChange={(e) => setUserEmail(e.target.value)}
              />
            </div>
          </div>
        )}

        {activeTab === "notifications" && (
          <div className="notifications-tab">
            <div className="notifications-header">
              <h2>Notifications</h2>
              <button onClick={loadNotifications}>
                Load Notifications
              </button>
            </div>

            {isLoadingNotifications && (
              <div className="loading-state">
                <p>Loading notifications...</p>
              </div>
            )}

            {notificationError && (
              <div className="error-state">
                <p>Error: {notificationError}</p>
                <button onClick={loadNotifications}>Retry</button>
              </div>
            )}

            {!isLoadingNotifications && !notificationError && notifications.length === 0 && (
              <p className="empty-state">
                No notifications. Click "Load Notifications" to fetch.
              </p>
            )}

            {!isLoadingNotifications && !notificationError && notifications.length > 0 && (
              <ul className="notifications-list">
                {notifications.map(notif => (
                  <li
                    key={notif.id}
                    className={notif.read ? "read" : "unread"}
                  >
                    <span>{notif.message}</span>
                    {!notif.read && (
                      <button onClick={() => markAsRead(notif.id)}>
                        Mark as Read
                      </button>
                    )}
                  </li>
                ))}
              </ul>
            )}
          </div>
        )}
      </main>
    </div>
  );
}

export default UserDashboard;
```

**Browser Behavior**:
- Click "Load Notifications"
- Shows "Loading notifications..." for 1.5 seconds
- 70% chance: Notifications appear
- 30% chance: Error message appears with "Retry" button

**Browser Console Output** (no errors, but you could add logging):
```
(Silent success or failure)
```

### The Loading-Error-Success Pattern

This is a fundamental pattern in React applications:

```tsx
const [data, setData] = useState(null);
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState(null);

// When starting an async operation:
setIsLoading(true);
setError(null);

// On success:
setData(result);
setIsLoading(false);

// On failure:
setError(errorMessage);
setIsLoading(false);

// In render:
if (isLoading) return <LoadingSpinner />;
if (error) return <ErrorMessage error={error} />;
if (!data) return <EmptyState />;
return <DataDisplay data={data} />;
```

### When to Apply: State Management Decision Framework

**Use separate useState calls when**:
- Each piece of state is independent
- State updates don't need to be synchronized
- You have fewer than ~5 related pieces of state

**Consider useReducer (Chapter 11) when**:
- You have complex state logic with multiple sub-values
- State updates depend on previous state
- You have many related pieces of state (>5)

**Consider lifting state up when**:
- Multiple components need the same state
- Components need to coordinate their state

**Consider Context (Chapter 11) when**:
- Many components at different nesting levels need the same state
- You're passing props through many layers (prop drilling)

**Consider external state management (Chapter 12-13) when**:
- State needs to persist across page navigation
- Multiple unrelated components need to share state
- You need advanced features like time-travel debugging

## Common Failure Modes and Their Signatures

## Common Failure Modes and Their Signatures

### Symptom: State update doesn't trigger re-render

**Browser behavior**:
You call a state setter, but the component doesn't re-render. The UI stays frozen.

**Console pattern**:
```
(No errors - state setter was called)
```

**DevTools clues**:
- React DevTools shows state didn't change
- Render count doesn't increase

**Root cause**: You're mutating state instead of creating a new value.

**Solution**: Always create new arrays/objects:

```tsx
// ❌ Wrong: Mutation
const [items, setItems] = useState([1, 2, 3]);
function addItem() {
  items.push(4); // Mutates array
  setItems(items); // Same reference
}

// ✅ Correct: New array
function addItem() {
  setItems([...items, 4]); // New array
}
```

### Symptom: Stale state in event handlers

**Browser behavior**:
Event handler uses old state value, even after state was updated.

**Console pattern**:
```
State updated to: 5
Event handler sees: 0
```

**Root cause**: Closure captured old state value.

**Solution**: Use functional state updates:

```tsx
// ❌ Wrong: Stale closure
const [count, setCount] = useState(0);

function handleClick() {
  setTimeout(() => {
    setCount(count + 1); // Uses stale count
  }, 1000);
}

// ✅ Correct: Functional update
function handleClick() {
  setTimeout(() => {
    setCount(prevCount => prevCount + 1); // Always uses latest
  }, 1000);
}
```

### Symptom: Too many re-renders

**Browser behavior**:
Browser freezes or becomes very slow. React error overlay appears.

**Console pattern**:
```
Error: Too many re-renders. React limits the number of renders to prevent an infinite loop.
```

**DevTools clues**:
- Profiler shows hundreds of renders per second
- Component render count increases rapidly

**Root cause**: State update in render body causes infinite loop.

**Solution**: Move state updates to event handlers or useEffect:

```tsx
// ❌ Wrong: State update during render
function Component() {
  const [count, setCount] = useState(0);
  setCount(count + 1); // Infinite loop!
  return <div>{count}</div>;
}

// ✅ Correct: State update in event handler
function Component() {
  const [count, setCount] = useState(0);
  return (
    <div>
      {count}
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}
```

### Symptom: Form input not updating

**Browser behavior**:
You type in an input, but nothing appears.

**Console pattern**:
```
Warning: You provided a `value` prop to a form field without an `onChange` handler.
```

**Root cause**: Controlled input without onChange handler.

**Solution**: Add onChange handler:

```tsx
// ❌ Wrong: No onChange
<input value={name} />

// ✅ Correct: With onChange
<input
  value={name}
  onChange={(e) => setName(e.target.value)}
/>
```

### Symptom: State resets unexpectedly

**Browser behavior**:
State resets to initial value when you don't expect it.

**Root cause**: Component is unmounting and remounting (key changed, or conditional rendering).

**Solution**: Lift state up to a parent that doesn't unmount, or use a state management library.

## The Journey: From Static to Interactive

## The Journey: From Static to Interactive

### The Complete Evolution

| Iteration | Problem | Technique Applied | Result | Key Insight |
|-----------|---------|-------------------|--------|-------------|
| 0 | Static UI with hardcoded values | None | No interactivity | Variables don't trigger re-renders |
| 1 | Need to update UI when data changes | `useState` | UI updates on button click | State + setter = re-render |
| 2 | Need user input | Event handlers + controlled inputs | Real-time input updates | `value` + `onChange` = controlled |
| 3 | Multiple form fields | Multiple `useState` calls | Complete editable form | Each state is independent |
| 4 | Complex UI with tabs | State for UI control | Multi-view dashboard | State controls what renders |
| 5 | Async operations | Loading/error state pattern | Robust data fetching | Three states: loading, error, success |

### Final Implementation: Production-Ready Dashboard

Here's our complete, production-ready dashboard incorporating all the patterns we've learned:

```tsx
// src/components/UserDashboard.tsx
import { useState } from 'react';

interface Notification {
  id: number;
  message: string;
  read: boolean;
  timestamp: Date;
}

function UserDashboard() {
  // UI state
  const [activeTab, setActiveTab] = useState<"profile" | "notifications">("profile");
  
  // Profile state
  const [userName, setUserName] = useState("Jane Smith");
  const [userEmail, setUserEmail] = useState("jane@example.com");
  const [userBio, setUserBio] = useState("Software developer");
  
  // Notifications state
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [isLoadingNotifications, setIsLoadingNotifications] = useState(false);
  const [notificationError, setNotificationError] = useState<string | null>(null);

  // Derived state
  const unreadCount = notifications.filter(n => !n.read).length;
  const hasNotifications = notifications.length > 0;

  // Profile handlers
  function handleProfileSubmit(event: React.FormEvent) {
    event.preventDefault();
    console.log("Profile saved:", { userName, userEmail, userBio });
    alert("Profile saved successfully!");
  }

  function handleProfileReset() {
    setUserName("Jane Smith");
    setUserEmail("jane@example.com");
    setUserBio("Software developer");
  }

  // Notification handlers
  function loadNotifications() {
    setIsLoadingNotifications(true);
    setNotificationError(null);

    // Simulate API call
    setTimeout(() => {
      if (Math.random() > 0.2) {
        setNotifications([
          {
            id: 1,
            message: "Welcome to your dashboard!",
            read: false,
            timestamp: new Date()
          },
          {
            id: 2,
            message: "Your profile is 80% complete",
            read: false,
            timestamp: new Date(Date.now() - 3600000)
          },
          {
            id: 3,
            message: "New message from support",
            read: true,
            timestamp: new Date(Date.now() - 7200000)
          }
        ]);
        setIsLoadingNotifications(false);
      } else {
        setNotificationError("Failed to load notifications. Please try again.");
        setIsLoadingNotifications(false);
      }
    }, 1500);
  }

  function markAsRead(notificationId: number) {
    setNotifications(notifications.map(notif =>
      notif.id === notificationId
        ? { ...notif, read: true }
        : notif
    ));
  }

  function markAllAsRead() {
    setNotifications(notifications.map(notif => ({ ...notif, read: true })));
  }

  function clearAllNotifications() {
    if (confirm("Clear all notifications?")) {
      setNotifications([]);
    }
  }

  return (
    <div className="dashboard">
      <header className="dashboard-header">
        <h1>User Dashboard</h1>
        <div className="user-info">
          <span className="user-name">{userName}</span>
          {unreadCount > 0 && (
            <span className="notification-badge" title={`${unreadCount} unread`}>
              {unreadCount}
            </span>
          )}
        </div>
      </header>

      <nav className="dashboard-tabs" role="tablist">
        <button
          role="tab"
          aria-selected={activeTab === "profile"}
          className={activeTab === "profile" ? "active" : ""}
          onClick={() => setActiveTab("profile")}
        >
          Profile
        </button>
        <button
          role="tab"
          aria-selected={activeTab === "notifications"}
          className={activeTab === "notifications" ? "active" : ""}
          onClick={() => setActiveTab("notifications")}
        >
          Notifications
          {unreadCount > 0 && (
            <span className="tab-badge">{unreadCount}</span>
          )}
        </button>
      </nav>

      <main className="dashboard-content">
        {activeTab === "profile" && (
          <div className="profile-tab" role="tabpanel">
            <h2>Profile Information</h2>
            <form onSubmit={handleProfileSubmit}>
              <div className="form-field">
                <label htmlFor="name">Name:</label>
                <input
                  id="name"
                  type="text"
                  value={userName}
                  onChange={(e) => setUserName(e.target.value)}
                  required
                />
              </div>

              <div className="form-field">
                <label htmlFor="email">Email:</label>
                <input
                  id="email"
                  type="email"
                  value={userEmail}
                  onChange={(e) => setUserEmail(e.target.value)}
                  required
                />
              </div>

              <div className="form-field">
                <label htmlFor="bio">Bio:</label>
                <textarea
                  id="bio"
                  value={userBio}
                  onChange={(e) => setUserBio(e.target.value)}
                  rows={4}
                  placeholder="Tell us about yourself..."
                />
              </div>

              <div className="form-actions">
                <button type="submit">Save Profile</button>
                <button type="button" onClick={handleProfileReset}>
                  Reset
                </button>
              </div>
            </form>
          </div>
        )}

        {activeTab === "notifications" && (
          <div className="notifications-tab" role="tabpanel">
            <div className="notifications-header">
              <h2>Notifications</h2>
              <div className="notifications-actions">
                {!hasNotifications && !isLoadingNotifications && (
                  <button onClick={loadNotifications}>
                    Load Notifications
                  </button>
                )}
                {hasNotifications && unreadCount > 0 && (
                  <button onClick={markAllAsRead}>
                    Mark All as Read
                  </button>
                )}
                {hasNotifications && (
                  <button onClick={clearAllNotifications}>
                    Clear All
                  </button>
                )}
              </div>
            </div>

            {isLoadingNotifications && (
              <div className="loading-state" role="status">
                <div className="spinner" aria-label="Loading"></div>
                <p>Loading notifications...</p>
              </div>
            )}

            {notificationError && (
              <div className="error-state" role="alert">
                <p className="error-message">{notificationError}</p>
                <button onClick={loadNotifications}>Retry</button>
              </div>
            )}

            {!isLoadingNotifications && !notificationError && !hasNotifications && (
              <div className="empty-state">
                <p>No notifications yet.</p>
                <p className="empty-state-hint">
                  Click "Load Notifications" to fetch your notifications.
                </p>
              </div>
            )}

            {!isLoadingNotifications && !notificationError && hasNotifications && (
              <ul className="notifications-list">
                {notifications.map(notif => (
                  <li
                    key={notif.id}
                    className={`notification-item ${notif.read ? "read" : "unread"}`}
                  >
                    <div className="notification-content">
                      <p className="notification-message">{notif.message}</p>
                      <time className="notification-time">
                        {notif.timestamp.toLocaleTimeString()}
                      </time>
                    </div>
                    {!notif.read && (
                      <button
                        className="mark-read-btn"
                        onClick={() => markAsRead(notif.id)}
                        aria-label={`Mark "${notif.message}" as read`}
                      >
                        Mark as Read
                      </button>
                    )}
                  </li>
                ))}
              </ul>
            )}
          </div>
        )}
      </main>
    </div>
  );
}

export default UserDashboard;
```

### Decision Framework: State Management Patterns

**When to use multiple useState calls**:
- ✅ Each piece of state is independent
- ✅ State updates don't need coordination
- ✅ You have fewer than 5-7 related pieces of state
- ✅ State logic is simple (direct updates)

**When to consider useReducer** (Chapter 11):
- ❌ State updates are complex (multiple related changes)
- ❌ Next state depends on previous state in complex ways
- ❌ You have many related pieces of state (>7)
- ❌ State transitions follow predictable patterns

**When to lift state up**:
- ❌ Multiple sibling components need the same state
- ❌ Parent needs to coordinate child components
- ❌ State needs to persist when child unmounts

**When to use derived state**:
- ✅ Value can be calculated from existing state
- ✅ Calculation is fast (not expensive)
- ✅ Value always stays in sync with source state

**When to store in separate state**:
- ❌ Value cannot be derived from existing state
- ❌ Value comes from user input or external source
- ❌ Value needs to persist independently

### Lessons Learned

**1. State is React's memory**
Regular variables reset on every render. State persists between renders and triggers re-renders when updated.

**2. Controlled components give you control**
By making React the single source of truth for input values, you can validate, transform, and synchronize data easily.

**3. State updates are asynchronous**
Don't rely on state values immediately after calling the setter. Use the next render or functional updates.

**4. Never mutate state**
Always create new arrays/objects. React compares references to detect changes.

**5. Derive when possible**
Don't store what you can calculate. Derived state can't get out of sync.

**6. Loading-error-success is fundamental**
Almost every async operation needs these three states. Make it a habit.

**7. Event handlers are your interface**
User interactions flow through event handlers. They're where you read user input and update state.

**8. State placement matters**
Keep state as local as possible. Only lift it up when multiple components need it.

### What's Next

We've built interactive UIs with local state, but we haven't dealt with side effects—operations that reach outside the component like data fetching, subscriptions, or DOM manipulation. In Chapter 4, we'll learn about `useEffect`, React's mechanism for handling side effects safely.

Our dashboard currently simulates API calls with `setTimeout`. In the next chapter, we'll replace that with real data fetching, and we'll discover why doing it wrong causes infinite loops, memory leaks, and race conditions—and how to do it right.
