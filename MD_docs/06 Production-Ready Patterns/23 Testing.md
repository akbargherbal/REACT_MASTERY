# Chapter 23: Testing

## Vitest: modern, fast testing

## The Confidence Problem

So far, we've built components by running our dev server and manually clicking around the browser. This works for a while, but it's slow, error-prone, and doesn't scale. How do you know that a change you made in the `UserProfile` component didn't accidentally break the `PaymentForm`? You don't. Not without manually re-testing every part of your application. This is where automated testing comes in. It's not about finding bugs; it's about building a system that gives you the confidence to refactor, add features, and ship code without fear.

In this chapter, we will build a safety net for our application. We'll start with the foundation—a test runner—and progressively build up to testing complex user interactions, API calls, and even entire user journeys.

### Phase 1: Establish the Reference Implementation

Our anchor example for this chapter will be a `FeedbackForm` component. It's a perfect candidate because it involves user input, state changes, and asynchronous operations (API calls).

Let's create the initial version. It will be simple: a text area and a disabled submit button. The button should only become enabled when the user has typed at least 10 characters.

**Project Structure**:
```
src/
└── components/
    └── FeedbackForm.tsx   ← Our reference component
```

Here is the initial implementation. We are deliberately not writing any tests for it yet. This is our starting point: a functional but untested component.

```tsx
// src/components/FeedbackForm.tsx

import { useState } from 'react';

const MIN_COMMENT_LENGTH = 10;

export function FeedbackForm() {
  const [comment, setComment] = useState('');

  const isSubmitDisabled = comment.length < MIN_COMMENT_LENGTH;

  const handleSubmit = (event: React.FormEvent<HTMLFormElement>) => {
    event.preventDefault();
    console.log('Submitting feedback:', comment);
    // API call would go here
  };

  return (
    <form onSubmit={handleSubmit}>
      <h2>Provide Feedback</h2>
      <label htmlFor="comment-input">Your feedback</label>
      <textarea
        id="comment-input"
        value={comment}
        onChange={(e) => setComment(e.target.value)}
        placeholder="Let us know what you think..."
      />
      <p>
        {comment.length}/{MIN_COMMENT_LENGTH} characters
      </p>
      <button type="submit" disabled={isSubmitDisabled}>
        Submit
      </button>
    </form>
  );
}
```

### The Need for a Test Runner

To test this component, we can't just run its file with Node.js. Why?
1.  It contains JSX, which Node can't parse.
2.  It uses browser-specific APIs like `document` and `window` (implicitly via React), which don't exist in Node.
3.  We need a framework to structure our tests, run them, and report the results.

This is the job of a **test runner**. For modern React/Next.js projects, **Vitest** is an excellent choice. It's built on top of Vite, making it incredibly fast, and it has a Jest-compatible API, which is the long-standing standard in the JavaScript world.

### Setting Up Vitest

Let's add Vitest to our project.

**Step 1: Install dependencies**

```bash
npm install -D vitest @vitejs/plugin-react jsdom
```

-   `vitest`: The test runner itself.
-   `@vitejs/plugin-react`: Allows Vitest to understand JSX and React-specific syntax.
-   `jsdom`: A JavaScript implementation of the DOM and browser APIs that lets us run our component tests in a simulated browser environment within Node.js.

**Step 2: Configure Vitest**

Create a `vitest.config.ts` file in the root of your project.

```typescript
// vitest.config.ts

import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
  },
});
```

-   `plugins: [react()]`: Tells Vitest to use the React plugin to process our files.
-   `environment: 'jsdom'`: This is the magic line. It tells Vitest to set up a `jsdom` environment before running our tests, so we have access to `document`, `window`, etc.
-   `globals: true`: This allows us to use Vitest APIs like `test`, `describe`, and `expect` in our test files without importing them, just like in Jest.

**Step 3: Add a test script to `package.json`**

```json
// package.json
{
  // ...
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "test": "vitest" // ← Add this line
  },
  // ...
}
```

### Iteration 0: The First Failing Test

Now, let's write our very first test. The goal is simple: does the component render at all without crashing?

Create a new file right next to our component. The `.test.tsx` convention is standard.

**Project Structure**:
```
src/
└── components/
    ├── FeedbackForm.tsx
    └── FeedbackForm.test.tsx   ← Our new test file
```

```tsx
// src/components/FeedbackForm.test.tsx

import { describe, test, expect } from 'vitest';
import { FeedbackForm } from './FeedbackForm';

describe('FeedbackForm', () => {
  test('should render', () => {
    // This is not how we'll actually test, but it's a start.
    // We need a way to render the component into our jsdom environment.
    const component = <FeedbackForm />;
    expect(component).toBeDefined();
  });
});
```

Let's run it:

```bash
npm test
```

The test passes! But it's a terrible test. We're not actually rendering the component or checking its output. We've just confirmed that we can create a React element. This doesn't tell us if the component works.

Let's try to actually render it. How would we do that? We need a library that can take our React component and render it into the `jsdom` document. This is the perfect entry point for React Testing Library.

Our current approach is insufficient because we have no way to inspect the DOM that our component *would* create. We need a tool to bridge the gap between our JSX and the simulated DOM provided by `jsdom`.

## React Testing Library: user-centric tests

## Testing from the User's Perspective

The biggest mistake developers make when testing UIs is testing implementation details. They might check if a component's internal state variable is `true`, or if a specific child component was rendered. This makes tests brittle. A small refactor that doesn't change the user's experience can break dozens of tests.

**React Testing Library (RTL)** offers a better philosophy:

> The more your tests resemble the way your software is used, the more confidence they can give you.

Instead of testing a component's internals, RTL forces you to ask questions from a user's perspective:
-   Can the user see the form title?
-   Can the user find the input field by its label?
-   When the user types in the field, does the character count update?
-   Is the submit button disabled initially?

This approach leads to resilient tests that verify actual user-facing behavior.

### Setting Up React Testing Library

**Step 1: Install dependencies**

```bash
npm install -D @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

-   `@testing-library/react`: Provides the core `render` function and queries to find elements in the DOM.
-   `@testing-library/jest-dom`: Adds custom matchers to `expect` like `.toBeInTheDocument()` and `.toBeDisabled()`, making assertions more readable.
-   `@testing-library/user-event`: A companion library for simulating user interactions like typing, clicking, and hovering, which is more robust than firing raw DOM events.

**Step 2: Configure Vitest to use `jest-dom`**

We need to tell Vitest to load the `jest-dom` matchers for every test. We do this with a setup file.

First, create the setup file:

```typescript
// src/setupTests.ts
import '@testing-library/jest-dom';
```

Next, update `vitest.config.ts` to use it:

```typescript
// vitest.config.ts

import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/setupTests.ts', // ← Add this line
  },
});
```

Now we're ready to write meaningful tests.

### Iteration 1: Testing the Initial Render

Let's rewrite our test to verify what the user sees when the form first loads.

**The Goal**:
1.  The heading "Provide Feedback" should be visible.
2.  The textarea should be identifiable by its label "Your feedback".
3.  The submit button should be present and disabled.

**Before** (Iteration 0):

```tsx
// src/components/FeedbackForm.test.tsx - Version 0
import { describe, test, expect } from 'vitest';
import { FeedbackForm } from './FeedbackForm';

describe('FeedbackForm', () => {
  test('should render', () => {
    const component = <FeedbackForm />;
    expect(component).toBeDefined();
  });
});
```

**After** (Iteration 1):

```tsx
// src/components/FeedbackForm.test.tsx - Version 1

import { render, screen } from '@testing-library/react';
import { describe, test, expect } from 'vitest';
import { FeedbackForm } from './FeedbackForm';

describe('FeedbackForm', () => {
  test('should render the form correctly', () => {
    // 1. Render the component into the jsdom
    render(<FeedbackForm />);

    // 2. Select elements like a user would
    const heading = screen.getByRole('heading', { name: /provide feedback/i });
    const commentInput = screen.getByLabelText(/your feedback/i);
    const submitButton = screen.getByRole('button', { name: /submit/i });

    // 3. Assert that the elements are in the document
    expect(heading).toBeInTheDocument();
    expect(commentInput).toBeInTheDocument();
    expect(submitButton).toBeInTheDocument();
    
    // 4. Assert specific attributes
    expect(submitButton).toBeDisabled();
  });
});
```

Let's break this down:
-   `render(<FeedbackForm />)`: This function from RTL renders our component into a container that is appended to `document.body`.
-   `screen`: This object provides a set of queries (`getBy...`, `findBy...`, `queryBy...`) that search the entire `document.body`.
-   `getByRole`: This is the preferred way to query. It finds elements based on their accessibility role. It's how a screen reader user would navigate your page. The `{ name: /.../i }` option finds an element with a specific accessible name (e.g., the text content of a button or heading). The `i` makes it case-insensitive.
-   `getByLabelText`: Finds an `<input>`, `<textarea>`, or `<select>` associated with a `<label>`.
-   `.toBeInTheDocument()` and `.toBeDisabled()`: These are the readable matchers from `@testing-library/jest-dom`.

Now, run the test:

```bash
npm test
```

**Terminal Output**:
```bash
 ✓ src/components/FeedbackForm.test.tsx (1)
   ✓ FeedbackForm (1)
     ✓ should render the form correctly

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  15:30:00
   Duration  258ms (transform 62ms, setup 85ms, collect 24ms, tests 5ms)

 PASS  Waiting for file changes...
       press h to show help, press q to quit
```
Success! We now have a test that gives us real confidence that our component renders correctly for the user on initial load.

### Iteration 2: Testing User Interaction

Our component isn't static. The core logic is that the submit button becomes enabled only after the user types 10 characters. Let's test that.

**The Goal**:
1.  Simulate a user typing less than 10 characters into the textarea.
2.  Assert the submit button is still disabled.
3.  Simulate the user typing more characters to meet the 10-character minimum.
4.  Assert the submit button is now enabled.

To do this, we'll use `@testing-library/user-event`. It provides a `setup` function that prepares an environment for simulating user interactions more accurately than firing manual events.

```tsx
// src/components/FeedbackForm.test.tsx - Add this new test case

import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event'; // Import user-event
import { describe, test, expect } from 'vitest';
import { FeedbackForm } from './FeedbackForm';

describe('FeedbackForm', () => {
  // ... our previous test is still here ...

  test('should enable the submit button only when feedback has enough characters', async () => {
    // The userEvent methods are async, so the test function must be async
    const user = userEvent.setup(); // Set up the user-event instance
    render(<FeedbackForm />);

    const commentInput = screen.getByLabelText(/your feedback/i);
    const submitButton = screen.getByRole('button', { name: /submit/i });

    // Initially, the button is disabled
    expect(submitButton).toBeDisabled();

    // 1. User types 9 characters
    await user.type(commentInput, 'Too short');
    
    // Assert the button is still disabled
    expect(submitButton).toBeDisabled();

    // 2. User types one more character
    await user.type(commentInput, '!'); // The input value is now "Too short!" (10 chars)

    // Assert the button is now enabled
    expect(submitButton).toBeEnabled();
  });
});
```

Let's run the tests again.

**Terminal Output**:
```bash
 ✓ src/components/FeedbackForm.test.tsx (2)
   ✓ FeedbackForm (2)
     ✓ should render the form correctly
     ✓ should enable the submit button only when feedback has enough characters

 Test Files  1 passed (1)
      Tests  2 passed (2)
   Start at  15:35:00
   Duration  310ms

 PASS  Waiting for file changes...
```
Perfect. We've now tested not just the static view, but the core interactive logic of our component, all from the user's perspective. We didn't need to inspect the `comment` state variable; we just checked its effect on the UI, which is exactly what a user would experience. This makes our test robust against refactoring.

## What to test (and what to skip)

## The Art of Prioritization

Now that we have the tools, a common question arises: "Do I have to test *everything*?" The answer is a firm **no**. Testing has a cost—it takes time to write and maintain tests. The goal is to maximize confidence for the minimum cost.

A helpful mental model is the **Testing Trophy**, popularized by Kent C. Dodds. It suggests a balanced portfolio of tests:

1.  **Static Analysis**: At the bottom, the largest and cheapest part, is static analysis. Tools like TypeScript and ESLint catch a huge class of errors before you even run your code. We've been using this all along.
2.  **Unit Tests**: These test a single module or component in isolation. They are fast and precise. Our `FeedbackForm` tests so far are unit tests.
3.  **Integration Tests**: These verify that several units work together correctly. For us, this often means testing a component that fetches data from a (mocked) API. We'll do this next.
4.  **End-to-End (E2E) Tests**: At the top, the smallest and most expensive part. These test your entire application in a real browser, from the UI to the database.

The Testing Trophy advises writing lots of static and unit/integration tests, and very few E2E tests.

### What You SHOULD Test

Focus your efforts on things that have logic and could break.

-   **User Interactions**: Anything that involves a user event (`click`, `type`, `submit`) and a resulting UI change. We just did this with our form's button logic.
-   **Conditional Rendering**: If a component can render in different ways based on props or state, test each case. Does the `LoadingSpinner` appear when `isLoading` is true? Does the `ErrorMessage` appear when `error` is not null?
-   **Accessibility**: RTL encourages this by default. Can you find all interactive elements by their accessible role and name? If not, your test is telling you your app has an accessibility issue.
-   **Complex Logic**: If a component contains complex business logic (e.g., a pricing calculator, a data visualization), write tests to cover its edge cases.
-   **API Communication**: Does your component correctly handle API success, loading, and error states? This is a prime candidate for integration testing.

### What You Can OFTEN Skip

Don't test things that have no logic or are the responsibility of another part of the system.

-   **Implementation Details**: **This is the golden rule.** Do not test the internal state of a component, its private methods, or which child components it renders. Test *what* the user sees, not *how* it's rendered.
    -   **Bad**: `expect(wrapper.state('comment')).toBe('hello')`
    -   **Good**: `expect(screen.getByLabelText(/your feedback/i)).toHaveValue('hello')`
-   **Third-Party Libraries**: Don't test that your charting library renders an SVG correctly or that `axios` sends an HTTP request. Trust that their authors have tested them. Your job is to test that you are *using them correctly*. This often involves mocking them.
-   **Static, Presentational Components**: A component that just takes props and renders some styled text or divs with no internal logic often doesn't need its own test file. Its presence will be implicitly tested by the parent component that uses it.
-   **Framework Internals**: Don't test that `useState` updates state or that `useEffect` runs on mount. Trust that React works. Test your application code.

Applying this to our `FeedbackForm`, we've correctly tested the interaction and conditional logic (the button's disabled state). The next logical step is to test its integration with an API.

## Integration tests that matter

## Testing Beyond the Component Bubble

Our `FeedbackForm` is meant to submit data to a server. A unit test can't verify this behavior because it runs in isolation. We need an **integration test** to verify the component's interaction with the network layer.

However, hitting a real network in our tests is a bad idea:
-   **Slow**: Network requests add seconds to test runs.
-   **Unreliable**: The test can fail due to network issues, not a code bug.
-   **Stateful**: Tests could pollute a real database.
-   **Difficult to test edge cases**: How do you force the server to return a 500 error to test your error handling?

The solution is to **mock the API**. We'll intercept outgoing network requests during our test run and provide a fake response, giving us full control over the "server."

### Introducing Mock Service Worker (MSW)

MSW is a powerful API mocking library. It works by intercepting requests at the network level, meaning your component code doesn't need to change at all. It thinks it's talking to a real server.

**Step 1: Install MSW**

```bash
npm install -D msw
```

**Step 2: Define Mock Handlers**

We'll create a central place for our mock API definitions.

```typescript
// src/mocks/handlers.ts

import { http, HttpResponse } from 'msw';

export const handlers = [
  // Handle POST requests to /api/feedback
  http.post('/api/feedback', async () => {
    // Respond with a 200 OK status and a JSON body.
    return HttpResponse.json({ success: true });
  }),
];
```

**Step 3: Create a Mock Server for Tests**

Now, we'll create a server instance that we can control in our tests.

```typescript
// src/mocks/server.ts

import { setupServer } from 'msw/node';
import { handlers } from './handlers';

// This configures a request mocking server with the given request handlers.
export const server = setupServer(...handlers);
```

**Step 4: Integrate MSW with Vitest**

We need to start the mock server before our tests run and shut it down after. We can update our test setup file for this.

```typescript
// src/setupTests.ts

import '@testing-library/jest-dom';
import { server } from './mocks/server';

// Establish API mocking before all tests.
beforeAll(() => server.listen());

// Reset any request handlers that we may add during tests,
// so they don't affect other tests.
afterEach(() => server.resetHandlers());

// Clean up after the tests are finished.
afterAll(() => server.close());
```

With this setup, any `fetch` request to `/api/feedback` during our tests will be intercepted by MSW.

### Iteration 3: Testing the Full Success Flow

First, let's update our `FeedbackForm` to handle submission logic, including loading and success states.

**Before** (Component without API logic):

```tsx
// src/components/FeedbackForm.tsx (Partial)
// ...
const handleSubmit = (event: React.FormEvent<HTMLFormElement>) => {
  event.preventDefault();
  console.log('Submitting feedback:', comment);
};
// ...
```

**After** (Component with API logic):

```tsx
// src/components/FeedbackForm.tsx (Updated)

import { useState } from 'react';

type FormStatus = 'idle' | 'loading' | 'success' | 'error';

const MIN_COMMENT_LENGTH = 10;

export function FeedbackForm() {
  const [comment, setComment] = useState('');
  const [status, setStatus] = useState<FormStatus>('idle');

  const isSubmitDisabled = comment.length < MIN_COMMENT_LENGTH || status === 'loading';

  const handleSubmit = async (event: React.FormEvent<HTMLFormElement>) => {
    event.preventDefault();
    setStatus('loading');
    try {
      const response = await fetch('/api/feedback', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ comment }),
      });
      if (!response.ok) {
        throw new Error('Server error');
      }
      setStatus('success');
    } catch (error) {
      setStatus('error');
    }
  };

  if (status === 'success') {
    return <h2>Thank you for your feedback!</h2>;
  }

  return (
    <form onSubmit={handleSubmit}>
      <h2>Provide Feedback</h2>
      {status === 'error' && <p style={{ color: 'red' }}>Something went wrong. Please try again.</p>}
      <label htmlFor="comment-input">Your feedback</label>
      <textarea
        id="comment-input"
        value={comment}
        onChange={(e) => setComment(e.target.value)}
        placeholder="Let us know what you think..."
      />
      <p>{comment.length}/{MIN_COMMENT_LENGTH} characters</p>
      <button type="submit" disabled={isSubmitDisabled}>
        {status === 'loading' ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

Now, let's write a test for the happy path.

**The Goal**:
1.  User types valid feedback.
2.  User clicks "Submit".
3.  The button text changes to "Submitting..." and becomes disabled.
4.  After the (mock) API call succeeds, the form is replaced with a "Thank you" message.

```tsx
// src/components/FeedbackForm.test.tsx - Add this new test case

// ... imports
import { http, HttpResponse } from 'msw';
import { server } from '../mocks/server';

describe('FeedbackForm', () => {
  // ... previous tests

  test('should show a success message after successful submission', async () => {
    const user = userEvent.setup();
    render(<FeedbackForm />);

    const commentInput = screen.getByLabelText(/your feedback/i);
    const submitButton = screen.getByRole('button', { name: /submit/i });

    // 1. User types valid feedback
    await user.type(commentInput, 'This is a great feature!');

    // 2. User clicks submit
    await user.click(submitButton);

    // 3. Assert loading state
    expect(screen.getByRole('button', { name: /submitting.../i })).toBeDisabled();

    // 4. Assert success message appears
    // `findBy` queries are async and wait for the element to appear
    const successMessage = await screen.findByText(/thank you for your feedback/i);
    expect(successMessage).toBeInTheDocument();

    // The form should be gone
    expect(submitButton).not.toBeInTheDocument();
  });
});
```

### Iteration 4: Testing the Error Handling Flow

What if the server fails? Our component should show an error message. With MSW, this is easy to test. We can override the mock handler for a single test.

**The Goal**:
1.  User types valid feedback and clicks "Submit".
2.  The (mock) API returns a 500 error.
3.  An error message appears.
4.  The submit button becomes enabled again, allowing the user to retry.

```tsx
// src/components/FeedbackForm.test.tsx - Add this new test case

// ... imports

describe('FeedbackForm', () => {
  // ... previous tests

  test('should show an error message if submission fails', async () => {
    // Override the default MSW handler for this specific test
    server.use(
      http.post('/api/feedback', () => {
        return new HttpResponse(null, { status: 500 });
      })
    );

    const user = userEvent.setup();
    render(<FeedbackForm />);

    const commentInput = screen.getByLabelText(/your feedback/i);
    const submitButton = screen.getByRole('button', { name: /submit/i });

    await user.type(commentInput, 'This is a test that will fail.');
    await user.click(submitButton);

    // Assert error message appears
    const errorMessage = await screen.findByText(/something went wrong/i);
    expect(errorMessage).toBeInTheDocument();

    // Assert the button is re-enabled and back to its original text
    expect(screen.getByRole('button', { name: /submit/i })).toBeEnabled();
  });
});
```

Run `npm test` one more time. All tests should pass. We now have a robust test suite for our component that covers its initial state, user interactions, and its integration with an external service, giving us extremely high confidence in its behavior.

## Playwright for E2E (when necessary)

## Leaving the Bubble: End-to-End Testing

Our Vitest/RTL/MSW setup is fantastic for testing components in isolation. We've verified our `FeedbackForm` works perfectly... assuming everything *around* it is also working.

-   What if we forgot to add the `<FeedbackForm />` component to our home page?
-   What if a global CSS rule accidentally hides the submit button?
-   What if the real API has a CORS issue that our mock server doesn't?

These are the types of problems that **End-to-End (E2E)** tests are designed to catch. An E2E test automates a real browser, navigating your live application just like a user would.

### Introducing Playwright

Playwright is a modern E2E testing framework from Microsoft. It's fast, reliable, and can run tests across Chromium (Chrome, Edge), Firefox, and WebKit (Safari).

### When to Write E2E Tests

E2E tests are at the top of the Testing Trophy for a reason: they are powerful but expensive.

-   **Slow**: They launch a real browser and interact with a running server, taking seconds or minutes to run, compared to milliseconds for unit tests.
-   **Brittle**: A small UI change (like renaming a button) can break them. They can also fail due to network flakiness.
-   **Costly**: They take longer to write and debug.

Therefore, **use E2E tests sparingly for your most critical user flows only.**

-   ✅ **Good candidates**: User registration, login, checkout process, core feature workflows.
-   ❌ **Bad candidates**: Testing every single form validation, visual styling, or simple component interactions. Use Vitest/RTL for those.

### Setting Up Playwright

Playwright has an interactive installer that makes setup easy.

```bash
npm init playwright@latest
```

Follow the prompts. It will:
1.  Install Playwright and browser binaries.
2.  Create a `playwright.config.ts` file.
3.  Create an example test file in a `tests` directory.

You'll want to adjust the `playwright.config.ts` to tell it how to run your Next.js dev server.

```typescript
// playwright.config.ts (Key parts)

import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  // ...
  webServer: {
    command: 'npm run dev',
    url: 'http://127.0.0.1:3000',
    reuseExistingServer: !process.env.CI,
  },
  use: {
    baseURL: 'http://127.0.0.1:3000',
    // ...
  },
  // ...
});
```

This `webServer` configuration tells Playwright to automatically start your Next.js dev server before running any tests.

### Writing an E2E Test for the Feedback Flow

Let's assume our `FeedbackForm` is rendered on the home page (`/`). We'll write a Playwright test to verify the entire submission flow against the real, running application.

**Project Structure**:
```
tests/
└── feedback.spec.ts   ← Our new E2E test
```

```typescript
// tests/feedback.spec.ts

import { test, expect } from '@playwright/test';

test('should allow a user to submit feedback', async ({ page }) => {
  // 1. Navigate to the page where the form is located
  await page.goto('/');

  // 2. Find elements and interact with them
  const commentInput = page.getByLabel('Your feedback');
  const submitButton = page.getByRole('button', { name: 'Submit' });

  // Assert initial state
  await expect(submitButton).toBeDisabled();

  // 3. Simulate user input
  await commentInput.fill('This is a real E2E test from a happy user!');

  // Assert button is now enabled
  await expect(submitButton).toBeEnabled();

  // 4. Click the submit button
  await submitButton.click();

  // 5. Assert the final state
  // Playwright's auto-waiting will handle the async nature of the API call.
  await expect(page.getByText('Thank you for your feedback!')).toBeVisible();
});
```

### Running the E2E Test

To run it, use the Playwright command line tool.

```bash
npx playwright test
```

Playwright will:
1.  Start your Next.js dev server.
2.  Launch a headless browser (Chromium by default).
3.  Navigate to your page and execute the steps in your test.
4.  Report the results and shut everything down.

This single test gives you confidence that your frontend, server, and API are all integrated correctly for this critical user path. It's the final, most comprehensive piece of your testing safety net.

### The Journey: From Problem to Solution

| Iteration | Failure Mode / Limitation                               | Technique Applied          | Result                                                              | Confidence Level |
| :-------- | :------------------------------------------------------ | :------------------------- | :------------------------------------------------------------------ | :--------------- |
| 0         | No automated verification; manual clicking required.    | **Vitest**                 | A test runner is in place, but tests are meaningless.               | Low              |
| 1         | Tests don't check what the user actually sees.          | **React Testing Library**  | Tests verify the initial rendered output from a user's perspective. | Medium           |
| 2         | Static render is tested, but not user interactions.     | **User Event**             | Core component logic (e.g., enabling a button) is verified.         | High             |
| 3         | Component is tested in a bubble, without API calls.     | **MSW (Mocking)**          | Component's integration with the network layer is verified.         | Very High        |
| 4         | No verification that the component works in the app.    | **Playwright (E2E)**       | The entire user flow is verified in a real browser.                 | Complete         |

### Final Decision Framework: Which Test Type When?

| Question                                              | Use This Tool                               | Example                                                              |
| ----------------------------------------------------- | ------------------------------------------- | -------------------------------------------------------------------- |
| Does my code have the correct types?                  | **TypeScript / ESLint** (Static)            | `const name: string = 123;` (Error)                                  |
| Does my single component render and behave correctly? | **Vitest + RTL + User Event** (Unit)        | Does clicking the "Show More" button reveal more text?               |
| Does my component handle API loading/error states?    | **Vitest + RTL + MSW** (Integration)        | Does a loading spinner show while fetching data?                     |
| Does a critical, multi-page user journey work?        | **Playwright** (E2E)                        | Can a user sign up, log in, and add an item to their cart?           |
| Am I using a third-party library?                     | **Don't test the library itself**           | Trust that `react-query` fetches data; test that your UI uses it correctly. |

By combining these layers, you build a robust, fast, and maintainable testing strategy that provides maximum confidence with minimum friction.
