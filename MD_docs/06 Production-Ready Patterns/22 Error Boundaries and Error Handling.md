# Chapter 22: Error Boundaries and Error Handling

## Error Boundaries in React

## The Fragility of a React App

In a complex React application, components are nested within other components, forming a tree-like structure. This composition is powerful, but it introduces a critical vulnerability: an unhandled error in a single, low-level component can crash the entire application.

Imagine a dashboard with a sidebar, a header, and a main content area. If the component responsible for rendering a user's avatar in the header throws an error, the entire dashboard—sidebar, header, and all—can unmount, leaving the user with a blank white screen. This is a catastrophic failure from a user experience perspective.

React's solution to this problem is the **Error Boundary**: a special component that can catch JavaScript errors anywhere in its child component tree, log those errors, and display a fallback UI instead of the component tree that crashed.

### Phase 1: Establish the Reference Implementation

Our anchor example will be a `UserWidget` that fetches and displays user data. We'll place it inside a `Dashboard` page that has other, stable UI elements. We will intentionally introduce a bug to demonstrate how it can take down the entire page.

**The Scenario**: Our API sometimes returns `null` for the `user` object if the user is not found. Our `UserWidget` component does not account for this and tries to access `user.name`, which will throw a `TypeError`.

**Project Structure**:

```
src/
└── app/
    └── dashboard/
        ├── components/
        │   └── UserWidget.tsx  <-- Our faulty component
        └── page.tsx            <-- The main dashboard page
```

Here is the initial, fragile implementation.

**The Faulty Component**:

```tsx
// src/app/dashboard/components/UserWidget.tsx

"use client";

import { useState, useEffect } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
}

// This function simulates fetching data from an API.
// We'll make it fail 50% of the time to test our error handling.
async function fetchUser(userId: string): Promise<User | null> {
  console.log("Fetching user data...");
  return new Promise((resolve) => {
    setTimeout(() => {
      if (Math.random() > 0.5) {
        resolve({
          id: userId,
          name: 'Jane Doe',
          email: 'jane.doe@example.com',
        });
      } else {
        // Simulate a "user not found" scenario
        resolve(null);
      }
    }, 1000);
  });
}

export default function UserWidget() {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetchUser('123').then(data => {
      setUser(data);
      setIsLoading(false);
    });
  }, []);

  if (isLoading) {
    return <div className="widget">Loading user data...</div>;
  }

  // THE BUG: We assume `user` is always an object, but it can be `null`.
  // This will throw "TypeError: Cannot read properties of null (reading 'name')"
  return (
    <div className="widget">
      <h3>User Profile</h3>
      <p>Name: {user.name}</p>
      <p>Email: {user.email}</p>
    </div>
  );
}
```

**The Dashboard Page**:

## Error Handling in Next.js

## The Next.js Way: `error.tsx`

While React's built-in Error Boundaries are powerful, the Next.js App Router provides a more integrated, file-based convention for handling errors within route segments: the `error.tsx` file.

An `error.tsx` file automatically creates a React Error Boundary that wraps a route segment and its nested children. It allows you to show granular, route-specific error UI and even provides a function to attempt recovery.

### Iteration 2: Handling Route Segment Errors with `error.tsx`

Let's refactor our dashboard to use the Next.js convention instead of our custom `ErrorBoundary` component.

**The Problem**: Our manual `ErrorBoundary` works, but it's verbose. We have to create the component, import it, and manually wrap the part of the UI we want to protect. Next.js can automate this for us at the route level.

**The Technique**: Create a file named `error.tsx` inside the `app/dashboard` directory. This file will define a client component that Next.js will automatically render in place of `page.tsx` if an error occurs during rendering on the server or client.

**Step 1: Create the `error.tsx` file.**

```tsx
// src/app/dashboard/error.tsx

"use client"; // Error components must be Client Components

import { useEffect } from 'react';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // You can log the error to an error reporting service here
    console.error(error);
  }, [error]);

  return (
    <div style={{ padding: '2rem' }}>
      <h2>Something went wrong in the dashboard!</h2>
      <p>{error.message}</p>
      <button
        onClick={
          // Attempt to recover by trying to re-render the segment
          () => reset()
        }
      >
        Try again
      </button>
    </div>
  );
}
```

**Key Features of `error.tsx`**:
-   It **must** be a Client Component (`"use client"`).
-   It receives two props:
    1.  `error`: A JavaScript `Error` object.
    2.  `reset`: A function that, when called, will attempt to re-render the contents of the Error Boundary. If successful, the fallback error component is replaced with the result of the re-render.

**Step 2: Remove the manual `ErrorBoundary` from `page.tsx`.**

Now that Next.js is handling this for us, we can simplify our page component.

**Before**:

## Logging and Monitoring (Sentry)

## From Reactive to Proactive: Production Error Monitoring

Displaying a friendly error message is good for the user, but it's a purely reactive measure. To build robust applications, we need to be proactive. We need to know when errors occur, how often, on what browsers, and what sequence of events led to the failure. This is where third-party error monitoring services like Sentry, LogRocket, or Datadog come in.

We'll use Sentry to demonstrate how to integrate a monitoring service into our Next.js application.

### Iteration 3: Integrating Sentry for Proactive Error Reporting

**The Problem**: We are blind to production errors. We rely on users to report bugs, which is unreliable and provides incomplete information.

**The Technique**: We will install and configure the Sentry Next.js SDK. It will automatically capture unhandled exceptions from our application (both client-side and server-side) and send detailed reports to the Sentry dashboard.

**Step 1: Install Sentry SDK.**

```bash
npm install --save @sentry/nextjs
```

**Step 2: Run the Sentry Wizard.**
The wizard will create and modify the necessary configuration files for you.

```bash
npx @sentry/wizard@latest -i nextjs
```

This command will:
1.  Log you into your Sentry account.
2.  Create `sentry.client.config.ts`, `sentry.server.config.ts`, and `sentry.edge.config.ts` files.
3.  Create a `.sentryclirc` file with your auth token.
4.  Update your `next.config.mjs` to automatically upload source maps during the build process. Source maps are crucial for seeing readable stack traces in Sentry.

**Step 3: Update `error.tsx` to report errors to Sentry.**
While Sentry automatically captures unhandled exceptions, it's good practice to explicitly capture errors handled by our boundaries to ensure all context is preserved.

```tsx
// src/app/dashboard/error.tsx (Updated with Sentry)

"use client";

import { useEffect } from 'react';
import * as Sentry from "@sentry/nextjs";

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Report the error to Sentry
    Sentry.captureException(error);
  }, [error]);

  return (
    <div style={{ padding: '2rem' }}>
      <h2>Something went wrong!</h2>
      <p>Our team has been notified of the issue. Please try again.</p>
      <button onClick={() => reset()}>
        Try again
      </button>
    </div>
  );
}
```

### Verification

Now, trigger the error in `UserWidget` one more time in a production-like environment (`npm run build` then `npm start`).

**Browser Behavior**:
The user experience is the same as before—they see the friendly error component.

**Sentry Dashboard**:
A new issue appears in your Sentry project dashboard. Clicking on it reveals a wealth of information:
-   **Error Message & Stack Trace**: `TypeError: Cannot read properties of null (reading 'name')` with a clean, de-minified stack trace pointing to `UserWidget.tsx`.
-   **Breadcrumbs**: A timeline of events that occurred before the error (e.g., user clicks, network requests). Our `console.log("Fetching user data...")` would appear here.
-   **Context**: Browser version, OS, device type, URL.
-   **Tags**: You can add custom tags like `userId` or `release version` to help filter and diagnose issues.

**Expected vs. Actual Improvement**:
- **Expected**: A way to know when errors happen in production.
- **Actual**: We now have a complete, automated system for capturing, aggregating, and analyzing production errors. We can fix bugs before most users even notice them. This moves us from a reactive debugging posture to a proactive one.

**Limitation Preview**: Our error UI is functional, but it could be much more helpful to the user. A generic "Something went wrong" message can be frustrating. How can we design an error state that is both reassuring and actionable?

## User-friendly error states

## Designing for Failure: Better Error Experiences

The final piece of the puzzle is the user-facing error UI itself. A well-designed error page can turn a moment of frustration into one of reassurance. It should acknowledge the problem, set expectations, and provide clear paths forward.

### Iteration 4: Creating a User-Centric Error Component

**The Problem**: Our current error message is generic and not very helpful. It doesn't give the user any control or useful information.

**The Technique**: We will redesign our `error.tsx` component based on best practices for user-friendly error states.

**Principles of Good Error UI**:
1.  **Be Human and Apologetic**: Use plain language. Acknowledge the failure and apologize for the inconvenience.
2.  **Explain What Happened (Briefly)**: Don't show a stack trace, but give a simple, high-level reason if possible (e.g., "We couldn't load your profile data.").
3.  **Provide Actionable Escapes**: Give the user clear things to do next. "Try again" is good. "Go to Homepage" is better. "Contact Support" is best for critical failures.
4.  **Offer a Reference ID**: If the user needs to contact support, giving them a unique ID for the error event makes the support process infinitely smoother. Sentry provides this out of the box.

**Step 1: Refactor `error.tsx` to be more user-friendly.**

```tsx
// src/app/dashboard/error.tsx (Final, User-Friendly Version)

"use client";

import { useEffect, useState } from 'react';
import * as Sentry from "@sentry/nextjs";
import Link from 'next/link';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  const [eventId, setEventId] = useState<string | null>(null);

  useEffect(() => {
    const id = Sentry.captureException(error);
    setEventId(id);
  }, [error]);

  return (
    <div style={{
      fontFamily: 'sans-serif',
      textAlign: 'center',
      padding: '4rem 2rem',
      border: '1px solid #e0e0e0',
      borderRadius: '8px',
      backgroundColor: '#fafafa',
      margin: '2rem'
    }}>
      <h2 style={{ fontSize: '2rem', color: '#c0392b' }}>Oops! Something went wrong.</h2>
      <p style={{ color: '#555', fontSize: '1.1rem' }}>
        We're sorry, but we encountered an unexpected error while trying to load this section.
        Our technical team has already been notified.
      </p>
      
      <div style={{ display: 'flex', justifyContent: 'center', gap: '1rem', marginTop: '2rem' }}>
        <button
          onClick={() => reset()}
          style={{
            padding: '0.75rem 1.5rem',
            fontSize: '1rem',
            cursor: 'pointer',
            backgroundColor: '#3498db',
            color: 'white',
            border: 'none',
            borderRadius: '4px'
          }}
        >
          Try Again
        </button>
        <Link href="/" style={{
            display: 'inline-block',
            padding: '0.75rem 1.5rem',
            fontSize: '1rem',
            backgroundColor: '#bdc3c7',
            color: '#2c3e50',
            textDecoration: 'none',
            borderRadius: '4px'
          }}>
          Go to Homepage
        </Link>
      </div>

      {eventId && (
        <p style={{ marginTop: '2rem', color: '#7f8c8d', fontSize: '0.9rem' }}>
          If the problem persists, please contact support and provide this error ID: <strong>{eventId}</strong>
        </p>
      )}
    </div>
  );
}
```

### Verification

Triggering the error now presents a much more polished and helpful UI.

**Browser Behavior**:
The user sees a friendly, well-designed message. They understand that the team is aware of the problem. They have two clear options: attempt to recover or navigate away to safety (the homepage). If they need to escalate, they have a specific error ID to give to a support agent, who can look it up directly in Sentry.

This final version transforms the error from a dead end into a managed incident.

### Common Failure Modes and Their Signatures

#### Symptom: Blank white screen in production.

**Browser behavior**: The entire page is blank. No content is rendered.
**Console pattern**: An error like `TypeError: ... is not a function` or `Cannot read properties of undefined`.
**DevTools clues**: The `<body>` tag is empty.
**Root cause**: An uncaught rendering error occurred, and no Error Boundary (`error.tsx` or `ErrorBoundary`) was in place to catch it.
**Solution**: Add an `error.tsx` file to the route segment where the error is occurring.

#### Symptom: Error boundary does not catch an error from an event handler.

**Browser behavior**: The UI is responsive, but clicking a button does nothing or causes a crash that isn't caught by the boundary.
**Console pattern**: An error message appears in the console, but the fallback UI does not render.
**DevTools clues**: The component tree remains intact.
**Root cause**: Error Boundaries only catch errors during rendering, in lifecycle methods, and in constructors. They do not catch errors inside event handlers.
**Solution**: Use a standard `try...catch` block inside the event handler function.

```tsx
function handleClick() {
  try {
    // Potentially failing logic
    doSomethingThatMightThrow();
  } catch (error) {
    console.error("Failed in handleClick", error);
    // Update state to show an error message to the user
    setErrorState("Something went wrong with that action.");
  }
}
```

### The Journey: From Problem to Solution

| Iteration | Failure Mode                               | Technique Applied          | Result                                                              | Developer Experience                               |
| :-------- | :----------------------------------------- | :------------------------- | :------------------------------------------------------------------ | :------------------------------------------------- |
| 0         | Entire app crashes on a `TypeError`.       | None                       | Blank screen; catastrophic failure.                                 | Debugging via console logs.                        |
| 1         | App crashes, but we want to contain it.    | React `ErrorBoundary`      | Only the faulty component is replaced; rest of the app is stable.   | Manual wrapping, class component boilerplate.      |
| 2         | Manual boundary is not Next.js idiomatic.  | Next.js `error.tsx`        | Route segment is replaced with error UI; includes `reset` function. | File-based convention, cleaner page code.          |
| 3         | We are blind to production errors.         | Sentry Integration         | Errors are automatically logged to a central dashboard.             | Proactive monitoring, rich context for debugging.  |
| 4         | Error UI is generic and unhelpful.         | User-Centric UI Design     | User gets clear info, actionable steps, and a support reference ID. | Confident that users have a good "off-ramp".       |

### Final Decision Framework: Which Error Handling Tool When?

| Scenario                               | Recommended Tool           | Why?                                                                                             |
| -------------------------------------- | -------------------------- | ------------------------------------------------------------------------------------------------ |
| **Error during component render**      | Next.js `error.tsx`        | The idiomatic, file-based convention for the App Router. Handles SSR and client errors.          |
| **Error in an event handler (e.g., `onClick`)** | `try...catch` block        | Error Boundaries do not catch errors in event handlers. This is the standard JS way.             |
| **Error in a `useEffect` or async code** | `try...catch` or `.catch()` | Asynchronous errors happen outside the render cycle and must be handled imperatively.            |
| **Error in the root layout**           | Next.js `global-error.tsx` | A safety net for the entire application when the root itself fails.                              |
| **Fine-grained control within a page** | React `ErrorBoundary`      | Use when you want to wrap a *specific component* inside a page, not the whole page/segment.      |
| **Production monitoring**              | Sentry (or similar)        | Essential for visibility into what's actually happening with your users. Complements all others. |
