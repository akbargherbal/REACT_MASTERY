# Chapter 16: Testing React Applications

## Testing Philosophy and Strategies

## Learning Objective

Understand the modern "Testing Trophy" philosophy as a guide for creating a balanced and effective testing strategy for a React application.

## Why This Matters

Writing tests isn't just about finding bugs; it's about building with confidence. A good testing strategy allows you to refactor code, add new features, and upgrade dependencies without fear of breaking existing functionality. The Testing Trophy provides a clear, modern roadmap for where to invest your testing efforts to get the best return on investment.

## Discovery Phase

For years, the "Testing Pyramid" was the dominant philosophy, emphasizing a large base of unit tests. However, in the world of component-based frameworks like React, this model can be misleading. A modern alternative, the **Testing Trophy**, proposed by Kent C. Dodds (the creator of React Testing Library), better reflects the needs of a modern web application.

Here's a breakdown of the Testing Trophy, from bottom to top:

```
      /================\
     |      E2E         |
    /====================\
   |    Integration     |
  /======================\
 |        Unit          |
/========================\
|        Static          |
\========================/
```

- **Static Analysis**: The foundation. Tools like TypeScript and ESLint catch typos, type errors, and basic code quality issues as you type, before you even run a test.
- **Unit Tests**: The next layer. These test a single, isolated piece of logic, like a helper function or a custom hook. They are fast and precise.
- **Integration Tests**: The sweet spot and the largest part of the trophy. These tests verify that multiple components work together correctly from the user's perspective. This is where React Testing Library shines.
- **End-to-End (E2E) Tests**: The top. These tests automate a real browser to simulate a complete user journey through your live application, including the backend. They provide the most confidence but are the slowest and most expensive to maintain.

The shape of the trophy—wide in the middle—suggests where you should spend the most time: **integration testing**.

## Deep Dive

### Why Not the Pyramid?

The classic pyramid advocates for a huge number of unit tests. In React, this can lead to "implementation detail" testing. For example, a unit test might check: "Does this component's state change from `false` to `true` when clicked?". This is brittle. If you refactor the component to use a different state management approach, the test breaks even if the user-facing behavior is identical.

### The Testing Trophy's Guiding Principle

The Testing Trophy encourages you to write tests that resemble how your users interact with your application. A user doesn't care about your component's state; they care that when they click a button, a modal appears. This is an integration of multiple units (the button, the state logic, the modal), and testing this flow provides much more value and confidence.

- **Static Analysis** gives you fast feedback.
- **Unit Tests** are for your core business logic (e.g., a function that calculates a shopping cart total).
- **Integration Tests** ensure your application works as a cohesive whole (e.g., adding an item to the cart updates the total correctly).
- **E2E Tests** are the final check that everything, including deployment and backend services, is working.

## Production Perspective

A balanced testing strategy based on the trophy gives you the highest "Return on Investment" (ROI).

- You spend the most time on **integration tests**, which give you high confidence that your app works for users, without being as slow or brittle as E2E tests.
- You use **unit tests** surgically for complex, isolated logic.
- You rely on **static analysis** to catch a whole class of errors for free.
- You use a few, critical **E2E tests** as a final safety net for your most important user flows (like login and checkout).

This module will primarily focus on the "Integration" and "Unit" layers, using Jest as our test runner and React Testing Library as our primary tool, perfectly aligning with the Testing Trophy philosophy.

## Jest Fundamentals

## Learning Objective

Understand the role of Jest as a test runner, assertion library, and mocking framework, and learn its basic syntax for writing a test.

## Why This Matters

To test your React components, you need a program that can find your test files, execute them, and report the results. Jest is the most popular test runner in the JavaScript ecosystem. It provides everything you need out of the box to start writing tests. Even if you use a modern alternative like Vitest, the syntax is nearly identical, making these fundamentals universally applicable.

## Discovery Phase

Let's write a simple test for a non-React helper function to understand Jest's core syntax. Imagine we have a utility function in our project.

```javascript
// src/utils/math.js
export const sum = (a, b) => a + b;
```

```javascript
// src/utils/math.test.js
import { sum } from "./math";

// 1. `describe` groups related tests together into a "test suite".
describe("sum", () => {
  // 2. `it` or `test` defines an individual test case.
  // The string should describe what the function is expected to do.
  it("should return the sum of two positive numbers", () => {
    // 3. `expect` is the assertion. You wrap the value you want to check.
    // 4. `.toBe` is a "matcher". It checks for strict equality (===).
    expect(sum(2, 3)).toBe(5);
  });

  it("should return the sum when one number is negative", () => {
    expect(sum(-5, 10)).toBe(5);
  });
});
```

When you run your test command (e.g., `npm test`), Jest will find this file (because it ends in `.test.js`), execute the code, and print a report to your console:

```
PASS  src/utils/math.test.js
  sum
    ✓ should return the sum of two positive numbers (2ms)
    ✓ should return the sum when one number is negative (1ms)

Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
```

## Deep Dive

Let's break down the key parts of a Jest test:

- **`describe(name, fn)`**: Creates a block that groups together several related tests. This is useful for organizing your tests into logical suites.
- **`it(name, fn)`** or **`test(name, fn)`**: These are interchangeable. This is the actual test case. The name should be a clear, human-readable description of the expected behavior.
- **`expect(value)`**: This is the heart of an assertion. You pass it the value that your code produced. It returns an "expectation" object.
- **Matchers**: You call a matcher function on the expectation object to assert something about the value. Jest has dozens of matchers:
  - `.toBe(value)`: Uses `Object.is` to test for exact equality. Great for primitives like numbers, strings, and booleans.
  - `.toEqual(value)`: Recursively checks every field of an object or array. Use this for testing objects and arrays.
  - `.toBeTruthy()` / `.toBeFalsy()`: Checks if a value is truthy or falsy.
  - `.toContain(item)`: Checks if an array or string contains an item.
  - `.toHaveBeenCalled()`: Checks if a mock function was called (we'll see this later).

### Common Confusion: `.toBe` vs. `.toEqual`

**You might think**: I can use `.toBe` to check if two objects are the same.

**Actually**: `.toBe` checks for referential equality (are they the exact same object in memory?), not structural equality (do they have the same properties?).

**Why the confusion happens**: It works for numbers and strings, so it's easy to assume it works for everything.

**Example**:

```javascript
it("should correctly compare objects", () => {
  const user1 = { name: "Alice" };
  const user2 = { name: "Alice" };

  // This will FAIL! user1 and user2 are different objects in memory.
  // expect(user1).toBe(user2);

  // This will PASS! .toEqual checks the contents of the objects.
  expect(user1).toEqual(user2);
});
```

**How to remember**: Use `.toBe` for primitives. Use `.toEqual` for objects and arrays.

## Production Perspective

- **Test Runner**: Jest discovers and runs your tests.
- **Assertion Library**: Jest provides `expect` and the matchers.
- **Mocking Framework**: Jest has powerful built-in functions for creating mock functions and modules (`jest.fn()`, `jest.spyOn()`, `jest.mock()`).
- **DOM Environment**: When testing React components, Jest uses a library called **JSDOM** to simulate a browser environment in Node.js, so you can "render" components and interact with a fake DOM without needing a real browser.

## React Testing Library

## Learning Objective

Use React Testing Library (RTL) to render components and write tests that find elements and assert their content, following the library's guiding principles.

## Why This Matters

React Testing Library has become the industry standard for testing React components. Its philosophy is to test your components in the same way a user would interact with them. This leads to tests that are more resilient to refactoring and give you more confidence that your application actually works for your users.

## Discovery Phase

Let's test a simple `Counter` component. It displays a count and a button to increment it.

```jsx
// src/components/Counter.jsx
import React, { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

```jsx
// src/components/Counter.test.jsx
import React from "react";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Counter } from "./Counter";

describe("Counter", () => {
  it("should render with an initial count of 0", () => {
    // 1. Render the component into the virtual DOM
    render(<Counter />);

    // 2. Use `screen` to find elements on the "page"
    // `getByText` finds an element by its text content.
    const countElement = screen.getByText("Count: 0");

    // 3. Assert that the element is in the document
    expect(countElement).toBeInTheDocument();
  });

  it("should increment the count when the button is clicked", async () => {
    // Arrange
    const user = userEvent.setup();
    render(<Counter />);

    // Act
    // `getByRole` is the preferred way to find interactive elements.
    const incrementButton = screen.getByRole("button", { name: /increment/i });
    await user.click(incrementButton);

    // Assert
    // The count should now be 1.
    const countElement = screen.getByText("Count: 1");
    expect(countElement).toBeInTheDocument();
  });
});
```

## Deep Dive

Let's break down the key parts of RTL:

1.  **`render`**: This function takes your JSX and renders it into a JSDOM container. You typically only call this once at the beginning of your test.

2.  **`screen`**: This is an object that has all the query methods pre-bound to the `document.body` of the JSDOM. It's the primary way you'll find elements.

3.  **Queries**: These are the functions you use to find elements. RTL encourages you to use queries that reflect how users find elements. The priority is:
    1.  **`getByRole`**: The best query. Finds elements by their accessibility role (e.g., `button`, `navigation`, `heading`). This aligns with how screen reader users navigate.
    2.  **`getByLabelText`**: Finds form elements by their associated `<label>`.
    3.  **`getByPlaceholderText`**: Finds form elements by their placeholder.
    4.  **`getByText`**: Finds elements by their text content.
    5.  **`getByTestId`**: The escape hatch. Use `data-testid="some-id"` when you can't find an element by any other means.

4.  **`userEvent`**: This library simulates real user interactions more accurately than RTL's built-in `fireEvent`. It dispatches all the events a browser would (e.g., `hover`, `focus`, `click`). Always prefer `userEvent` for interaction tests. Note that its methods are async, so you should `await` them.

### RTL's Guiding Principle: Avoid Testing Implementation Details

**You might think**: I should test that the `count` state inside the component is `1`.

**Actually**: You should never test a component's internal state. This is an **implementation detail**.

**Why the confusion happens**: It feels natural to want to check the internal workings of your component.

**How to remember**: The user cannot see your component's state. They only see the rendered output. If you refactor your `Counter` to use the `useReducer` hook instead of `useState`, the user experience is identical. A test that checks the state would break, but a test that checks the text on the screen (`Count: 1`) would still pass. This is what makes RTL tests so robust and maintainable. **Test the behavior, not the implementation.**

## Production Perspective

- **Jest + RTL + userEvent**: This trio forms the modern standard for component testing in React.
- **`@testing-library/jest-dom`**: This companion library (usually set up for you by default) adds custom matchers to Jest like `.toBeInTheDocument()`, `.toBeVisible()`, and `.toHaveValue()`, which make your assertions more declarative and readable.
- **Accessibility (`a11y`)**: By prioritizing queries like `getByRole` and `getByLabelText`, RTL gently pushes you to write more accessible HTML. If your component is hard to test with these queries, it's often a sign that it's also hard for users with assistive technologies to use.

## Testing Components with Actions

## Learning Objective

Write integration tests for components that use React Router's `<Form>` and route `action` functionality to handle data mutations.

## Why This Matters

React 19's Actions paradigm (covered in Chapter 8) moves form submission logic out of the component and into the router configuration. Testing this requires a special setup where you provide a test router to your component, allowing you to verify that the form correctly calls its associated action with the right data.

## Discovery Phase

Let's test a simple contact form that uses a route action.

```jsx
// src/components/ContactForm.jsx
import React from "react";
import { Form, useNavigation } from "react-router-dom";

export function ContactForm() {
  const navigation = useNavigation();
  const isSubmitting = navigation.state === "submitting";

  return (
    <Form method="post">
      <label htmlFor="email">Email</label>
      <input id="email" name="email" type="email" />

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Submitting..." : "Submit"}
      </button>
    </Form>
  );
}
```

```jsx
// src/components/ContactForm.test.jsx
import React from "react";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { createMemoryRouter, RouterProvider } from "react-router-dom";
import { ContactForm } from "./ContactForm";

describe("ContactForm", () => {
  it("should call the action with the form data on submission", async () => {
    const user = userEvent.setup();

    // 1. Create a mock action function using jest.fn()
    const mockAction = jest.fn();

    // 2. Define a test route configuration
    const testRoutes = [
      {
        path: "/contact",
        element: <ContactForm />,
        action: mockAction, // Assign the mock action to the route
      },
    ];

    // 3. Create a memory router for the test
    const router = createMemoryRouter(testRoutes, {
      initialEntries: ["/contact"],
    });

    // 4. Render the component wrapped in the RouterProvider
    render(<RouterProvider router={router} />);

    // 5. Interact with the form
    const emailInput = screen.getByLabelText(/email/i);
    await user.type(emailInput, "test@example.com");

    const submitButton = screen.getByRole("button", { name: /submit/i });
    await user.click(submitButton);

    // 6. Assert that the mock action was called
    expect(mockAction).toHaveBeenCalledTimes(1);

    // Optional: Assert what it was called with (more advanced)
    const request = mockAction.mock.calls.request;
    const formData = await request.formData();
    expect(formData.get("email")).toBe("test@example.com");
  });
});
```

## Deep Dive

Let's trace the test's execution, as the setup is more complex than a simple component render.

1.  **`jest.fn()`**: We create a "spy" or mock function. This function doesn't do anything, but it keeps a record of every time it was called, and what arguments it was called with. This is the key to our assertion.

2.  **`createMemoryRouter`**: A standard browser has a real router that uses the URL bar. For our tests in a JSDOM environment, we need a simulated router. `createMemoryRouter` creates an in-memory router that doesn't rely on a browser history. We provide it our test route configuration.

3.  **`<RouterProvider>`**: Just like in our main application, this component provides the routing context that components like `<Form>` need to function.

4.  **Interaction**: `userEvent` simulates typing into the input and clicking the submit button.

5.  **The Magic**: When the button is clicked, React Router's `<Form>` component prevents a real form submission. Instead, it finds the action for the current route (`/contact`) in our test router, which is our `mockAction`. It then calls `mockAction` with a `Request` object containing the form data.

6.  **Assertion**: Our test's final step is simple: we ask our mock function, "Were you called?". `expect(mockAction).toHaveBeenCalledTimes(1)` verifies that the entire chain of events worked correctly. We can even go further and inspect the `Request` object to ensure the data was correct.

## Production Perspective

- **Separation of Concerns**: This testing pattern perfectly mirrors the architectural benefit of Actions. The component test is responsible for verifying that the UI is wired up correctly and that submitting the form triggers the action.
- **Unit Testing Actions**: The action logic itself (e.g., calling an API, validating data) can be tested separately as a pure function, without needing to render any React components. This leads to faster, more focused tests.
- **Confidence**: This integration test gives you high confidence that your form works from the user's perspective. It confirms that the inputs have the correct `name` attributes and that the submission process is correctly handled by React Router.

## Testing Server Components

## Learning Objective

Write a unit test for a React Server Component (RSC) to verify its rendered output based on props and mocked data.

## Why This Matters

Server Components run in a different environment (Node.js or an edge runtime) and have different capabilities than client components. They cannot be tested in the same way because they are not interactive and don't run in a browser context. The strategy is to test them as asynchronous functions that return renderable JSX.

## Discovery Phase

Let's test a simple RSC that fetches user data and displays their name.

```jsx
// src/components/UserProfile.jsx (A Server Component)
import React from "react";
import { fetchUser } from "../api/user"; // A data fetching function

export async function UserProfile({ userId }) {
  const user = await fetchUser(userId);

  if (!user) {
    return <div>User not found.</div>;
  }

  return (
    <div>
      <h1>{user.name}</h1>
      <p>Email: {user.email}</p>
    </div>
  );
}
```

```jsx
// src/components/UserProfile.test.jsx
import React from "react";
import { render, screen } from "@testing-library/react";
import { UserProfile } from "./UserProfile";
import { fetchUser } from "../api/user";

// 1. Mock the entire data fetching module
jest.mock("../api/user");

describe("UserProfile", () => {
  it("should render the user name and email after fetching data", async () => {
    // 2. Define the mock data for this specific test
    const mockUser = { id: 1, name: "John Doe", email: "john.doe@example.com" };

    // Configure the mock function to return our data
    fetchUser.mockResolvedValue(mockUser);

    // 3. Await the Server Component function to get the JSX promise
    const jsx = await UserProfile({ userId: 1 });

    // 4. Render the resulting JSX using RTL
    render(jsx);

    // 5. Assert the content is in the document
    expect(
      screen.getByRole("heading", { name: /john doe/i }),
    ).toBeInTheDocument();
    expect(screen.getByText(/john.doe@example.com/i)).toBeInTheDocument();
  });

  it('should render a "not found" message if no user is returned', async () => {
    // Configure the mock to return null for this test case
    fetchUser.mockResolvedValue(null);

    const jsx = await UserProfile({ userId: 99 });
    render(jsx);

    expect(screen.getByText(/user not found/i)).toBeInTheDocument();
  });
});
```

## Deep Dive

This testing model is fundamentally different from testing a client component.

1.  **Mocking Dependencies**: Server Components often have server-side dependencies, like database clients or API fetchers. In our test, we must mock `fetchUser`. `jest.mock('../api/user')` replaces the actual module with a mock version, and `fetchUser.mockResolvedValue(mockUser)` tells it to return a promise that resolves to our fake user data.

2.  **`async`/`await` the Component**: A Server Component that fetches data is an `async` function. In our test, we can just call it like any other async function: `await UserProfile({ userId: 1 })`. This executes the component's logic in the Node.js environment provided by Jest.

3.  **The Result is JSX**: The promise returned by the RSC resolves to the JSX it's supposed to render. It's not HTML yet, just the React element tree.

4.  **Render and Assert**: Once we have the JSX, we can pass it to RTL's `render` function. This renders the JSX to JSDOM, and from there we can use `screen` queries to assert that the correct content was rendered, just like with any other component.

### Common Confusion: "Where is `userEvent`? Why don't I test clicks?"

**You might think**: I need to test user interactions with this component.

**Actually**: Server Components have **no interactivity**. They cannot have `onClick` handlers or `useState`. Their sole job is to fetch data and produce a UI.

**Why the confusion happens**: It's a React component, so our instinct is to test it like one.

**How to remember**: Test a Server Component like you would test a template engine. You provide it with inputs (props and mocked data), and you assert that it produces the correct output (the rendered JSX). Any interactivity would be handled by Client Components that you might import into your RSC, and those would be tested separately using `userEvent`.

## Production Perspective

- **Speed**: This testing method is extremely fast. It doesn't need to simulate a full component lifecycle with re-renders. It's just an async function call.
- **Isolation**: It allows you to test the data-rendering logic of your RSCs in complete isolation from the client-side of your application.
- **Focus**: This forces you to test what the RSC is actually responsible for: correctly transforming data into a UI.

## Testing Hooks

## Learning Objective

Isolate and test the logic of a custom hook using the `renderHook` utility from React Testing Library.

## Why This Matters

Custom hooks are the primary way to extract and reuse stateful logic in modern React. This logic is often complex and critical to your application's functionality. Testing a hook in isolation, without needing to render a full component, results in simpler, faster, and more focused tests of your core business logic.

## Discovery Phase

Let's create and test a simple `useToggle` custom hook.

```jsx
// src/hooks/useToggle.js
import { useState, useCallback } from "react";

export function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => {
    setValue((v) => !v);
  }, []);

  return [value, toggle];
}
```

```jsx
// src/hooks/useToggle.test.js
import { renderHook, act } from "@testing-library/react";
import { useToggle } from "./useToggle";

describe("useToggle", () => {
  it("should initialize with the default value of false", () => {
    // 1. Render the hook using renderHook
    const { result } = renderHook(() => useToggle());

    // 2. Access the hook's return value via `result.current`
    expect(result.current).toBe(false);
  });

  it("should initialize with a provided initial value", () => {
    const { result } = renderHook(() => useToggle(true));
    expect(result.current).toBe(true);
  });

  it("should toggle the value when the toggle function is called", () => {
    const { result } = renderHook(() => useToggle());

    // 3. Wrap any action that causes a state update in `act()`
    act(() => {
      // result.current is our `toggle` function
      result.current();
    });

    // 4. Assert that the value has changed
    expect(result.current).toBe(true);

    // Toggle it back
    act(() => {
      result.current();
    });
    expect(result.current).toBe(false);
  });
});
```

## Deep Dive

### `renderHook`

You can't call a hook from a regular JavaScript function; it must be called from within a React component. The `renderHook` utility does exactly this for you. It creates a tiny, invisible test harness component, calls your hook inside of it, and gives you access to its return value.

The object returned by `renderHook` contains a few properties, but the most important one is `result`.

- **`result.current`**: This property always holds the most recent return value of your hook. When the hook's state updates and it "re-renders" within the test harness, `result.current` will be updated to reflect the new return value.

### `act()`

React updates state asynchronously in batches. In a test environment, we need a way to ensure that all state updates and effects have been processed before we make our assertions. The `act` utility does this.

**Rule of thumb**: Any code that triggers a state update in your hook (e.g., calling a function it returns) must be wrapped in `act()`.

When you use `userEvent` to test a full component, it automatically wraps its actions in `act` for you. But when testing a hook in isolation, you have to do it manually.

The test flow is:

1.  `renderHook` to get the initial state.
2.  Assert the initial state.
3.  Wrap a state-updating call in `act()`.
4.  Assert the new state reflected in `result.current`.

## Production Perspective

- **Logic vs. UI**: Testing hooks allows you to completely separate the testing of your business logic from the testing of your UI. This aligns perfectly with the purpose of hooks themselves.
- **Speed and Simplicity**: A hook test is a unit test. It's much faster and simpler than a full component integration test. For a complex piece of logic, it's far easier to test all the edge cases in a hook test than by trying to simulate them through component interactions.
- **Refactoring Confidence**: If you have thorough tests for your custom hooks, you can confidently refactor the components that use them, knowing that the underlying logic is solid.

## Testing Async Behavior

## Learning Objective

Write tests for components that fetch data, correctly handling and asserting loading, success, and error states using asynchronous queries.

## Why This Matters

A huge portion of a modern web application involves fetching data from an API. Your components need to handle the entire lifecycle of these requests: showing a loading indicator, displaying the data when it arrives, and showing an error message if it fails. Your tests must be able to verify all three of these states to be complete.

## Discovery Phase

Let's test a component that fetches and displays a user's name from an API.

```jsx
// src/components/UserData.jsx
import React, { useState, useEffect } from "react";

export function UserData({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    fetch(`https://api.example.com/users/${userId}`)
      .then((res) => {
        if (!res.ok) throw new Error("Failed to fetch");
        return res.json();
      })
      .then((data) => setUser(data))
      .catch((err) => setError(err.message))
      .finally(() => setLoading(false));
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return null;

  return <h1>Welcome, {user.name}</h1>;
}
```

```jsx
// src/components/UserData.test.jsx
import React from "react";
import { render, screen } from "@testing-library/react";
import { UserData } from "./UserData";

// Mock the global fetch function
global.fetch = jest.fn();

describe("UserData", () => {
  beforeEach(() => {
    // Clear mock history before each test
    fetch.mockClear();
  });

  it("should show a loading message initially and then display the user name", async () => {
    const mockUser = { name: "Alice" };
    fetch.mockResolvedValueOnce({
      ok: true,
      json: async () => mockUser,
    });

    render(<UserData userId={1} />);

    // 1. Assert the initial loading state
    expect(screen.getByText("Loading...")).toBeInTheDocument();

    // 2. Use `findBy` to wait for the success state to appear
    const heading = await screen.findByRole("heading", {
      name: /welcome, alice/i,
    });
    expect(heading).toBeInTheDocument();

    // 3. Assert that the loading message is gone
    expect(screen.queryByText("Loading...")).not.toBeInTheDocument();
  });

  it("should display an error message if the fetch fails", async () => {
    // Mock a failed network request
    fetch.mockRejectedValueOnce(new Error("Network error"));

    render(<UserData userId={2} />);

    // Wait for and assert the error message
    const errorMessage = await screen.findByText(/error/i);
    expect(errorMessage).toBeInTheDocument();
  });
});
```

## Deep Dive

### Mocking `fetch`

The first step in any async test is to take control of the asynchronous process. We don't want our tests to make real network requests. They would be slow, unreliable, and dependent on an external service. By mocking `global.fetch`, we can instantly resolve or reject the fetch promise with whatever data we want, allowing us to test different scenarios predictably.

### `getBy` vs. `findBy` vs. `queryBy`

RTL provides three families of queries for different situations.

- **`getBy...`**: Finds an element or throws an error immediately if it's not found. Use this for asserting elements you expect to be on the screen right after a render. (e.g., the initial "Loading..." message).
- **`queryBy...`**: Finds an element or returns `null` if it's not found. It does not throw an error. Use this for asserting that an element is **not** on the screen. (e.g., `expect(screen.queryByText('Loading...')).not.toBeInTheDocument()`).
- **`findBy...`**: Returns a promise that resolves when the element is found. If the element is not found after a default timeout (usually 1000ms), the promise rejects and the test fails. This is the **primary tool for testing asynchronous UI updates**.

When we `await screen.findByRole('heading', ...)` RTL will repeatedly check the DOM until the heading appears or the timeout is reached. This is how we wait for our `fetch` promise to resolve and our component to re-render with the new data.

### `waitFor`

Sometimes you need to wait for something to happen that isn't just an element appearing. The `waitFor` utility can be used for this. It takes an async callback and waits until the assertions inside it pass.

```javascript
// Example of using waitFor
await waitFor(() => {
  expect(myMockFunction).toHaveBeenCalledTimes(1);
});
```

## Production Perspective

- **Mock Service Worker (MSW)**: While mocking `fetch` globally is fine for simple tests, it can become cumbersome in a large application. **Mock Service Worker (MSW)** is a library that allows you to define a mock API server that intercepts network requests at the network level. This is a more robust and scalable approach, as you can define your mock API once and reuse it across all your tests, and even during development.
- **Testing All States**: A robust test suite for an async component will always have at least three tests: one for the loading state, one for the success state, and one for the error state. This ensures you've handled all possible outcomes of the asynchronous operation.

## Integration Testing

## Learning Objective

Write an integration test that simulates a complete user flow across multiple components and routes, verifying that they work together as expected.

## Why This Matters

Unit tests are great for verifying individual pieces, but they don't guarantee that the pieces fit together correctly. Integration tests provide this guarantee. By testing a complete user journey, you gain a high degree of confidence that your application is working from the user's perspective. This is the largest and most valuable part of the "Testing Trophy".

## Discovery Phase

Let's test a simple multi-page application. A user starts on a home page with a link to a "Users" page. Clicking the link should navigate them to a list of users.

```jsx
// src/App.jsx (A simplified app with routing)
import React from "react";
import {
  Link,
  Outlet,
  createMemoryRouter,
  RouterProvider,
} from "react-router-dom";

const HomePage = () => (
  <div>
    <h1>Home</h1>
    <Link to="/users">View Users</Link>
  </div>
);
const UsersPage = () => (
  <div>
    <h1>Users</h1>
    <ul>
      <li>Alice</li>
      <li>Bob</li>
    </ul>
  </div>
);

const routerConfig = [
  { path: "/", element: <HomePage /> },
  { path: "/users", element: <UsersPage /> },
];

// We export the router creation to use in our test
export const createAppRouter = (initialEntries = ["/"]) => {
  return createMemoryRouter(routerConfig, { initialEntries });
};

// The main App component
export const App = () => {
  const router = createAppRouter();
  return <RouterProvider router={router} />;
};
```

```jsx
// src/App.test.jsx
import React from "react";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { RouterProvider } from "react-router-dom";
import { createAppRouter } from "./App";

describe("App Navigation Flow", () => {
  it("should navigate from the home page to the users page", async () => {
    const user = userEvent.setup();

    // 1. Create a router instance for the test
    const router = createAppRouter(["/"]); // Start at the home page

    // 2. Render the application with the test router
    render(<RouterProvider router={router} />);

    // 3. Verify we are on the home page
    expect(screen.getByRole("heading", { name: /home/i })).toBeInTheDocument();

    // 4. Simulate the user clicking the link
    const usersLink = screen.getByRole("link", { name: /view users/i });
    await user.click(usersLink);

    // 5. Wait for the new page to appear and assert its content
    const usersHeading = await screen.findByRole("heading", { name: /users/i });
    expect(usersHeading).toBeInTheDocument();
    expect(screen.getByText("Alice")).toBeInTheDocument();
  });
});
```

## Deep Dive

This test combines many of the techniques we've already learned into a single, powerful flow.

- **Test Setup**: Just like in our Actions test, we use `createMemoryRouter` to provide a routing context. This is essential for any test that involves navigation. We render the entire application wrapped in the `<RouterProvider>`.
- **Arrange, Act, Assert**: The test follows the classic AAA pattern.
  - **Arrange**: We set up the router and render the application. We assert our starting condition (we're on the home page).
  - **Act**: The user performs an action: `await user.click(usersLink)`.
  - **Assert**: We assert the outcome. The URL changes, and the `UsersPage` component should be rendered. We use `findByRole` to wait for the asynchronous navigation and re-render to complete.

### The Power of Black-Box Testing

Notice what this test _doesn't_ know:

- It doesn't know about the `HomePage` or `UsersPage` components.
- It doesn't know that clicking the link changes the URL.
- It doesn't know how the router works internally.

The test behaves exactly like a user: it finds a link with certain text, clicks it, and expects to see new content on the screen. This is a form of **black-box testing**. Because the test is not coupled to the implementation details, you could completely refactor your routing setup or component structure, and as long as the user flow remains the same, the test will continue to pass.

## Production Perspective

- **Critical User Flows**: In a real application, you would write integration tests for your most important user journeys.
  - Can a user successfully log in and see their dashboard?
  - Can a user add a product to the cart and complete the checkout process?
  - Can a user create, edit, and delete a post?
- **Confidence vs. Cost**: These tests provide the highest level of confidence outside of full E2E tests. They are slower to run than unit tests because they render more components, but the value they provide in verifying that your application works as a cohesive whole is immense. A good test suite has a healthy number of these core integration tests.

## End-to-End Testing with Playwright

## Learning Objective

Understand the purpose of End-to-End (E2E) testing and how it differs from integration testing with React Testing Library.

## Why This Matters

The Testing Trophy is topped with E2E tests for a reason. While RTL integration tests give you confidence that your React code works together, they run in a simulated environment. E2E tests are the ultimate verification: they run your _entire_, fully-built application in a _real_ browser, interacting with your _real_ backend. They are your final line of defense against bugs that only appear in a production-like environment.

## Discovery Phase

Let's look at what a simple login test might look like using **Playwright**, a popular E2E testing framework. This code would live in a separate test suite, outside of your main Jest/RTL setup.

```javascript
// tests/login.spec.js (This is a Playwright test file)
import { test, expect } from "@playwright/test";

test.describe("Login Flow", () => {
  test("should allow a user to log in and see the dashboard", async ({
    page,
  }) => {
    // 1. Navigate to the running application's login page
    await page.goto("http://localhost:3000/login");

    // 2. Find elements and interact with them using browser-level selectors
    await page.getByLabel("Email").fill("test@example.com");
    await page.getByLabel("Password").fill("password123");
    await page.getByRole("button", { name: "Log In" }).click();

    // 3. Wait for navigation and assert the new page's content
    // This will wait for the URL to change to '/dashboard'
    await page.waitForURL("http://localhost:3000/dashboard");

    // Assert that an element on the dashboard is visible
    const welcomeMessage = page.getByRole("heading", { name: /welcome/i });
    await expect(welcomeMessage).toBeVisible();
  });
});
```

## Deep Dive

### RTL vs. Playwright: Key Differences

| Feature         | React Testing Library (Integration)                            | Playwright (End-to-End)                                                    |
| :-------------- | :------------------------------------------------------------- | :------------------------------------------------------------------------- |
| **Environment** | Node.js + JSDOM (a simulated browser)                          | A real browser (Chrome, Firefox, Safari)                                   |
| **Scope**       | Tests your React components in isolation from the backend.     | Tests your entire application stack (frontend + backend + database).       |
| **Execution**   | Runs your source code directly.                                | Interacts with your fully built, running application via HTTP.             |
| **Speed**       | Fast (milliseconds to seconds per test).                       | Slow (seconds to minutes per test).                                        |
| **Assertions**  | Asserts that components render correctly and state is managed. | Asserts that real user journeys work across the entire system.             |
| **Debugging**   | Debug in your code editor and console.                         | Provides powerful debugging tools like video recordings and trace viewers. |

### Why Do You Need Both?

- **RTL Integration Tests** are for verifying that your React application's UI logic is correct. They are fast and you can write many of them. They answer the question: "Did I build the thing right?"
- **E2E Tests** are for verifying that the entire system is integrated correctly. They catch problems that RTL can't, such as:
  - Backend API errors or contract mismatches.
  - Environment configuration issues.
  - CSS bugs that only appear in a real browser rendering engine.
  - Authentication and database problems.

E2E tests answer the question: "Did I build the right thing, and does it actually work in the real world?"

## Production Perspective

- **The Tip of the Trophy**: E2E tests are powerful but also the most expensive to write, run, and maintain. They can be "flaky" (failing intermittently due to network issues or timing). For this reason, you should have only a few of them.
- **Critical Paths Only**: Reserve E2E tests for your application's most critical, "money-making" paths.
  - User registration and login.
  - The core checkout or purchase flow.
  - The main feature that defines your product.
- **Smoke Tests**: E2E tests are often used as "smoke tests" after a deployment. You run a quick suite of E2E tests against your newly deployed production environment to get a signal that nothing is fundamentally broken.
- **Frameworks**: **Playwright** and **Cypress** are the two leading frameworks in this space. Both are excellent choices for modern E2E testing.

## Test-Driven Development with React

## Learning Objective

Apply the Test-Driven Development (TDD) workflow of "Red, Green, Refactor" to build a React component.

## Why This Matters

Test-Driven Development is a methodology that flips the usual development process on its head. Instead of writing your component and then writing tests for it, you write a failing test first, then write the component code to make it pass. This process can lead to better-designed, more testable components and ensures that every line of code you write is covered by a test.

## Discovery Phase

Let's build a simple `Accordion` component using the TDD workflow. It should display a title, and when the title is clicked, it should reveal some content.

### Step 1: Red - Write a Failing Test

Our first requirement is that the content should not be visible by default.

```jsx
// src/components/Accordion.test.jsx
import React from "react";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Accordion } from "./Accordion"; // This file doesn't exist yet!

describe("Accordion", () => {
  it("should not show the content by default", () => {
    render(<Accordion title="Test Title" content="Some content" />);

    // `queryByText` returns null if not found, so this is perfect for asserting absence.
    expect(screen.queryByText("Some content")).not.toBeInTheDocument();
  });
});
```

**Run the test.** It will fail spectacularly because `Accordion.jsx` doesn't even exist. This is the "Red" step.

### Step 2: Green - Make the Test Pass

Now, we write the _absolute minimum_ amount of code to make this test pass.

```jsx
// src/components/Accordion.jsx
import React from "react";

export function Accordion({ title, content }) {
  // We don't even need state yet. Just don't render the content.
  return (
    <div>
      <button>{title}</button>
    </div>
  );
}
```

**Run the test again.** It now passes! The content isn't rendered, so `queryByText` returns `null`. This is the "Green" step.

### Step 3: Red - Write the Next Failing Test

Our next requirement is that clicking the title reveals the content.

```jsx
// Add this test to Accordion.test.jsx
it("should show the content after clicking the title", async () => {
  const user = userEvent.setup();
  render(<Accordion title="Test Title" content="Some content" />);

  const titleButton = screen.getByRole("button", { name: /test title/i });
  await user.click(titleButton);

  // This will fail because our component has no logic to show the content.
  expect(screen.getByText("Some content")).toBeInTheDocument();
});
```

**Run the tests.** The first test still passes, but our new test fails. We are back to "Red".

### Step 4: Green - Make It Pass Again

Now we add the logic to our component to handle the click.

```jsx
// src/components/Accordion.jsx
import React, { useState } from "react";

export function Accordion({ title, content }) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>{title}</button>
      {isOpen && <div>{content}</div>}
    </div>
  );
}
```

**Run the tests.** Both tests now pass! We are "Green" again.

### Step 5: Refactor

Our component is simple, so there isn't much to refactor. But at this stage, you could clean up variable names, extract sub-components, or improve styling, all while continuously running your tests to ensure you haven't broken anything. This cycle—Red, Green, Refactor—repeats for every new feature you add to the component.

## Deep Dive

The TDD cycle is a powerful feedback loop:

- **Red**: Writing the test first forces you to think from the user's perspective. What should this component do? What is its public API (its props)? It clearly defines the "what" before you think about the "how".
- **Green**: Writing the simplest code to pass the test prevents over-engineering. You only write code in response to a failing test, so you only write code that is strictly necessary.
- **Refactor**: With a safety net of passing tests, you are free to improve the implementation details of your component without fear.

## Production Perspective

- **A Discipline, Not a Dogma**: Not every developer practices pure TDD for every component. However, the mindset it encourages is invaluable. Even if you write your tests just after you write your code (a practice called "Test-After Development"), the process of thinking about how to test a component often reveals flaws in its design.
- **Component Design**: TDD often leads to better-designed components. If a component is hard to test, it's often a sign that it's doing too much. The TDD process encourages you to break down complex components into smaller, more focused, and more testable units.
- **Confidence and Coverage**: By definition, TDD leads to 100% test coverage for the features you've built. This provides a robust safety net for future development and refactoring.

## Module Synthesis 📋

## Module Synthesis: Building with Confidence

In this chapter, we've built a comprehensive understanding of how to test a modern React application. We moved beyond the simple idea of "checking for bugs" and embraced testing as a core part of the development process that enables confidence, maintainability, and better software design.

We started with a solid foundation in **testing philosophy**, adopting the **Testing Trophy** as our guide to balancing different types of tests for the best return on investment. We established our toolchain with **Jest** as the test runner and learned its fundamental assertion syntax.

The core of the chapter focused on **React Testing Library**, internalizing its user-centric philosophy. We learned to test everything from simple components to complex asynchronous behavior, user interactions, and the new **Actions** pattern in React 19.

We then demystified testing for modern React architectures, creating clear strategies for testing both **Server Components** (as async functions) and **custom hooks** (in isolation with `renderHook`).

Finally, we zoomed out to see the bigger picture, understanding the role of full **End-to-End tests** with tools like Playwright as the ultimate system-wide verification, and we explored the **Test-Driven Development** workflow as a discipline for building robust components from the ground up.

## Looking Ahead

A well-tested application is a prerequisite for long-term success and scalability. The skills you've learned here will be invaluable as we move into the final parts of the course.

- In **Chapter 17: Server-Side Rendering and Frameworks**, the testing patterns for Server and Client components will become even more relevant as we build full-stack applications with frameworks like Next.js.
- In **Chapter 28: Security Best Practices**, we'll see how testing can be a first line of defense against common vulnerabilities.

You are no longer just a developer who can build features; you are a developer who can build reliable, maintainable, and production-ready applications, backed by the confidence that only a solid test suite can provide.
