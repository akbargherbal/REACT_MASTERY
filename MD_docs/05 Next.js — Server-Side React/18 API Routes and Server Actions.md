# Chapter 18: API Routes and Server Actions

## Building API endpoints

## The Server-Side Connection

So far, our applications have been primarily self-contained on the client. But real-world applications need to persist data, communicate with databases, and perform secure operations. This requires a backend—a server-side component that our React components can talk to.

In Next.js, you don't need a separate Express or Node.js server. The framework provides two powerful mechanisms for building your backend logic directly within your project: **API Routes** and **Server Actions**.

This chapter will build a simple "Guestbook" application to explore both approaches. We'll see how each one works, where they shine, and how to handle real-world challenges like form submissions, data validation, and error handling.

### Phase 1: Establish the Reference Implementation

Our anchor example is a Guestbook. Users can enter their name and a message, submit it, and see it appear in a list of entries.

Let's start by building the UI components and a mock database.

**Project Structure**:
```
src/
├── app/
│   ├── page.tsx              <- Our main page with the form and list
│   └── layout.tsx
├── lib/
│   └── db.ts                 <- A simple in-memory "database"
└── components/
    └── GuestbookForm.tsx     <- The form component
```

First, our mock database. This will just be an in-memory array to keep things simple. In a real app, this would connect to a service like Postgres, MongoDB, or Supabase.

```typescript
// src/lib/db.ts

export interface GuestbookEntry {
  id: string;
  name: string;
  message: string;
  createdAt: Date;
}

// In-memory array to act as our database
const entries: GuestbookEntry[] = [
  {
    id: '1',
    name: 'Ada Lovelace',
    message: 'The Analytical Engine has no pretensions whatever to originate anything.',
    createdAt: new Date(),
  },
];

// Simulate database operations
export const db = {
  getEntries: async (): Promise<GuestbookEntry[]> => {
    // Sort by most recent
    return [...entries].sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
  },
  addEntry: async (entry: { name: string; message: string }): Promise<GuestbookEntry> => {
    const newEntry: GuestbookEntry = {
      id: (entries.length + 1).toString(),
      ...entry,
      createdAt: new Date(),
    };
    entries.push(newEntry);
    return newEntry;
  },
};
```

Next, the main page component, which will fetch and display the entries. We'll make this a Server Component to fetch data directly.

```tsx
// src/app/page.tsx

import { db, GuestbookEntry } from '@/lib/db';
import { GuestbookForm } from '@/components/GuestbookForm';

async function GuestbookEntriesList() {
  const entries = await db.getEntries();

  return (
    <div className="mt-8 space-y-4">
      <h2 className="text-2xl font-semibold">Entries</h2>
      {entries.map((entry) => (
        <div key={entry.id} className="p-4 border rounded-lg bg-gray-50">
          <p className="font-semibold">{entry.name}</p>
          <p className="text-gray-700">{entry.message}</p>
          <p className="text-xs text-gray-400 mt-2">
            {entry.createdAt.toLocaleString()}
          </p>
        </div>
      ))}
    </div>
  );
}

export default function GuestbookPage() {
  return (
    <main className="max-w-2xl mx-auto p-8">
      <h1 className="text-4xl font-bold">Guestbook</h1>
      <p className="text-gray-600 mt-2">
        Leave a message for future visitors.
      </p>
      <GuestbookForm />
      <GuestbookEntriesList />
    </main>
  );
}
```

Finally, our form component. This must be a Client Component (`"use client"`) because it uses `useState` and handles user interaction. For now, its `onSubmit` handler will just log the form data to the console. This is our starting point—a UI with no backend connection.

```tsx
// src/components/GuestbookForm.tsx

'use client';

import { useState, FormEvent } from 'react';

export function GuestbookForm() {
  const [name, setName] = useState('');
  const [message, setMessage] = useState('');

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    console.log('Form submitted with:', { name, message });
    // How do we send this to the server?
  };

  return (
    <form onSubmit={handleSubmit} className="mt-6 space-y-4">
      <div>
        <label htmlFor="name" className="block text-sm font-medium text-gray-700">
          Name
        </label>
        <input
          id="name"
          type="text"
          value={name}
          onChange={(e) => setName(e.target.value)}
          className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
          required
        />
      </div>
      <div>
        <label htmlFor="message" className="block text-sm font-medium text-gray-700">
          Message
        </label>
        <textarea
          id="message"
          value={message}
          onChange={(e) => setMessage(e.target.value)}
          rows={3}
          className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
          required
        />
      </div>
      <button
        type="submit"
        className="inline-flex justify-center py-2 px-4 border border-transparent shadow-sm text-sm font-medium rounded-md text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
      >
        Sign Guestbook
      </button>
    </form>
  );
}
```

### Iteration 1: Connecting the Form with an API Route

Our form is isolated on the client. It collects data, but has no way to send it to the server to be saved in our "database". This is the classic use case for an API endpoint.

In Next.js App Router, you create API endpoints by creating a `route.ts` (or `.js`) file inside a folder within the `app` directory. The folder path determines the URL. For a `/api/guestbook` endpoint, we create `app/api/guestbook/route.ts`.

Inside this file, you export named functions corresponding to HTTP methods: `GET`, `POST`, `PUT`, `DELETE`, etc.

Let's create a `POST` handler to receive our guestbook data.

**Project Structure Update**:
```
src/
└── app/
    ├── api/
    │   └── guestbook/
    │       └── route.ts      <- Our new API endpoint
    └── page.tsx
```

```typescript
// src/app/api/guestbook/route.ts

import { NextResponse } from 'next/server';
import { db } from '@/lib/db';

export async function POST(request: Request) {
  try {
    // 1. Parse the incoming request body
    const body = await request.json();
    const { name, message } = body;

    // Basic validation
    if (!name || !message) {
      return NextResponse.json(
        { error: 'Name and message are required' },
        { status: 400 }
      );
    }

    // 2. Add the entry to the database
    const newEntry = await db.addEntry({ name, message });

    // 3. Return a success response
    return NextResponse.json(newEntry, { status: 201 });
  } catch (error) {
    console.error('Failed to create guestbook entry:', error);
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    );
  }
}
```

Now we have a server endpoint waiting for requests at `POST /api/guestbook`. Let's update our `GuestbookForm` to send its data there using the `fetch` API.

We also need a way to tell the `GuestbookEntriesList` to refresh after a new entry is added. A simple way is to use `router.refresh()` from `next/navigation`, which re-fetches data for the current route (in our case, re-running the Server Component that calls `db.getEntries()`).

**Before** (Iteration 0):

```tsx
// src/components/GuestbookForm.tsx (snippet)
const handleSubmit = async (e: FormEvent) => {
  e.preventDefault();
  console.log('Form submitted with:', { name, message });
  // How do we send this to the server?
};
```

**After** (Iteration 1):

```tsx
// src/components/GuestbookForm.tsx (updated)

'use client';

import { useState, FormEvent } from 'react';
import { useRouter } from 'next/navigation';

export function GuestbookForm() {
  const router = useRouter();
  const [name, setName] = useState('');
  const [message, setMessage] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setIsLoading(true);
    setError(null);

    try {
      const response = await fetch('/api/guestbook', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ name, message }),
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.error || 'Something went wrong');
      }

      // Clear form
      setName('');
      setMessage('');

      // Refresh the page to show the new entry
      router.refresh();

    } catch (err) {
      setError(err instanceof Error ? err.message : 'An unknown error occurred');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="mt-6 space-y-4">
      {/* ... form inputs ... */}
      <div>
        <label htmlFor="name">...</label>
        <input id="name" value={name} onChange={(e) => setName(e.target.value)} />
      </div>
      <div>
        <label htmlFor="message">...</label>
        <textarea id="message" value={message} onChange={(e) => setMessage(e.target.value)} />
      </div>
      
      <button
        type="submit"
        disabled={isLoading}
        className="inline-flex justify-center py-2 px-4 border border-transparent shadow-sm text-sm font-medium rounded-md text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500 disabled:opacity-50"
      >
        {isLoading ? 'Submitting...' : 'Sign Guestbook'}
      </button>
      {error && <p className="text-red-500 mt-2">{error}</p>}
    </form>
  );
}
```

### Verification: Checking the Network

Let's run the app and submit the form.

**Browser Behavior**:
The user fills out the form, clicks "Sign Guestbook". The button text changes to "Submitting...". After a moment, the form clears, the button becomes active again, and the new entry appears at the top of the list without a full page reload.

**Browser DevTools - Network Tab**:
```
- Request URL: http://localhost:3000/api/guestbook
- Request Method: POST
- Status Code: 201 Created
- Request Payload: {"name":"Charlie","message":"Hello, world!"}
- Response: {"id":"2","name":"Charlie","message":"Hello, world!", ...}
```

**Let's parse this evidence**:
1.  **What the user experiences**: A smooth, interactive form submission. The UI provides feedback (loading state) and updates automatically.
2.  **What the network reveals**: Our client-side `fetch` call correctly sent a `POST` request to our API route. The server processed it and returned a `201 Created` status, confirming success.
3.  **Root cause identified**: We have successfully bridged the client-server gap. The client component captures user input, and the API route provides a server-side handler to process and persist that data.

This is the traditional, battle-tested way of handling data mutations in web applications. It's explicit, follows standard REST principles, and works well. However, notice the amount of client-side boilerplate we had to write: `useState` for loading and error states, the `fetch` call with its headers and body, JSON stringification, and response parsing.

**Limitation preview**: This works, but it's a lot of manual work on the client. Could we simplify this? What if we could just call a server function directly from our component, without all the `fetch` ceremony?

## Server Actions: mutations without API routes

## A More Direct Approach

The API Route approach works, but it creates a separation. You have client-side logic for *calling* the API and server-side logic for *implementing* the API. This often leads to boilerplate code on the client to manage the state of the request (loading, error, success).

Server Actions, introduced with the App Router, offer a more integrated model. They allow you to define a function on the server that can be called directly from your client components as if it were a regular JavaScript function. Next.js handles the underlying network request for you.

### Iteration 2: Refactoring to a Server Action

Let's refactor our Guestbook to use a Server Action instead of an API Route.

**Step 1: Define the Action**

Server Actions are asynchronous functions defined with the `"use server";` directive at the top of the function body or the top of the file. It's common practice to keep them in a separate `actions.ts` file.

**Project Structure Update**:
```
src/
├── app/
│   ├── actions.ts            <- Our new Server Action file
│   └── page.tsx
├── lib/
│   └── db.ts
└── components/
    └── GuestbookForm.tsx
```

```typescript
// src/app/actions.ts

'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function addEntry(data: FormData) {
  const name = data.get('name') as string;
  const message = data.get('message') as string;

  // Basic validation
  if (!name || !message) {
    throw new Error('Name and message are required');
  }

  await db.addEntry({ name, message });

  // Revalidate the page to show the new entry
  revalidatePath('/');
}
```

A few key things to note here:
1.  `'use server';`: This directive marks this function (and all others in the file) as code that should only ever run on the server.
2.  `data: FormData`: Server Actions invoked from a `<form>` naturally receive a `FormData` object, which is a standard web API for representing form data.
3.  `revalidatePath('/')`: This is the Server Action equivalent of `router.refresh()`. It tells Next.js to purge the cache for the specified path (`/`), forcing a re-fetch of the data for that page on the next visit. This is how our list of entries will update.

**Step 2: Update the Form Component**

Now, we can dramatically simplify our `GuestbookForm` component. We no longer need `useState` for loading/error, `fetch`, or `useRouter`. We can just pass our server action directly to the `<form>`'s `action` prop.

**Before** (Iteration 1 - API Route):

```tsx
// src/components/GuestbookForm.tsx (API Route version)

'use client';

import { useState, FormEvent } from 'react';
import { useRouter } from 'next/navigation';

export function GuestbookForm() {
  const router = useRouter();
  const [name, setName] = useState('');
  const [message, setMessage] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setIsLoading(true);
    setError(null);

    try {
      const response = await fetch('/api/guestbook', { /* ... fetch options ... */ });
      // ... error handling, router.refresh(), etc. ...
    } catch (err) {
      // ... catch block ...
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="mt-6 space-y-4">
      {/* ... inputs and button with disabled state ... */}
    </form>
  );
}
```

**After** (Iteration 2 - Server Action):

```tsx
// src/components/GuestbookForm.tsx (Server Action version)

'use client';

import { useRef } from 'react';
import { addEntry } from '@/app/actions';

export function GuestbookForm() {
  const formRef = useRef<HTMLFormElement>(null);

  return (
    <form
      ref={formRef}
      action={async (formData) => {
        await addEntry(formData);
        formRef.current?.reset();
      }}
      className="mt-6 space-y-4"
    >
      <div>
        <label htmlFor="name" className="block text-sm font-medium text-gray-700">
          Name
        </label>
        <input
          id="name"
          name="name" // The 'name' attribute is crucial for FormData
          type="text"
          className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
          required
        />
      </div>
      <div>
        <label htmlFor="message" className="block text-sm font-medium text-gray-700">
          Message
        </label>
        <textarea
          id="message"
          name="message" // The 'name' attribute is crucial for FormData
          rows={3}
          className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
          required
        />
      </div>
      <button
        type="submit"
        className="inline-flex justify-center py-2 px-4 border border-transparent shadow-sm text-sm font-medium rounded-md text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
      >
        Sign Guestbook
      </button>
    </form>
  );
}
```

### Analysis: What Just Happened?

The client component is now incredibly simple.
- No `useState` for name/message (uncontrolled component).
- No `useState` for loading/error.
- No `useEffect` or `useRouter`.
- No `fetch` call.

We pass the server function `addEntry` to the `action` prop of the form. When the form is submitted, Next.js:
1.  Intercepts the form submission.
2.  Serializes the form data.
3.  Sends a `POST` request to a special, auto-generated endpoint on the server.
4.  Invokes our `addEntry` Server Action with the form data.
5.  Handles the pending UI state automatically (more on this later).

**Verification**:
Run the app. The form works identically to the user. But if you inspect the network tab, you'll see a `POST` request, not to `/api/guestbook`, but to the current page URL (`/`). The request payload is no longer JSON but `FormData`. Next.js is managing this RPC (Remote Procedure Call) for us.

**Improvement**:
The amount of client-side code has been reduced by over 50%. We've removed manual state management for the request lifecycle and eliminated the need to create and maintain a separate API route file. The logic for mutating data is now co-located with the data fetching (`revalidatePath`) in a single server-side function.

**Limitation preview**: Our form is sleek and simple, but it relies entirely on client-side JavaScript to function. What happens if a user has JavaScript disabled or is on a very slow network where the JS hasn't loaded yet?

## Form handling with progressive enhancement

## The Power of the Platform

One of the most significant advantages of Server Actions is their ability to work seamlessly with the fundamentals of the web platform, specifically HTML forms. This enables **progressive enhancement**: building a baseline experience that works without JavaScript, which is then enhanced when JavaScript becomes available.

### Iteration 3: Demonstrating the Failure

Let's see what happens to our current Server Action implementation if JavaScript is disabled.

**How to Simulate**:
1.  Open your browser's Developer Tools.
2.  In Chrome/Edge, press `Ctrl+Shift+P` (or `Cmd+Shift+P` on Mac) to open the Command Menu.
3.  Type "Disable JavaScript" and press Enter.
4.  Refresh the page.

Now, try to submit the guestbook form.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
You fill out the form and click "Sign Guestbook". Absolutely nothing happens. The form remains filled, the button does nothing. It's completely broken.

**Browser Console Output**:
```
(empty - no JavaScript is running to log anything)
```

**Network Tab Analysis**:
```
(empty - no request is made when the button is clicked)
```

**Let's parse this evidence**:

1.  **What the user experiences**: A dead form. It looks interactive but is non-functional.
2.  **What the console reveals**: Nothing, which is the key clue. Our event handlers aren't firing.
3.  **Root cause identified**: Our `<form action={...}>` relies on a client-side JavaScript wrapper to intercept the submit event and trigger the Server Action RPC call. Without JavaScript, the form has no default submission behavior and does nothing.
4.  **Why the current approach can't solve this**: It's fundamentally a JS-driven solution.
5.  **What we need**: A way to tell the browser where to send the form data using standard HTML, so it works even without client-side scripting.

### Iteration 4: Achieving Progressive Enhancement

The fix is surprisingly simple. Instead of passing an async function to the `action` prop, we just pass the Server Action *itself*.

**Before** (Iteration 2 - JS-dependent action):

```tsx
// src/components/GuestbookForm.tsx (snippet)

// ...
const formRef = useRef<HTMLFormElement>(null);

return (
  <form
    ref={formRef}
    action={async (formData) => {
      await addEntry(formData);
      formRef.current?.reset(); // This part requires JS
    }}
    // ...
  >
    {/* ... */}
  </form>
);
```

**After** (Iteration 3 - Progressively Enhanced):

```tsx
// src/components/GuestbookForm.tsx (updated)

'use client';

// We no longer need useRef for this basic version
import { addEntry } from '@/app/actions';

export function GuestbookForm() {
  // Note: We lose the ability to reset the form on the client
  // without JS. We will address state handling in the next section.
  return (
    <form action={addEntry} className="mt-6 space-y-4">
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" name="name" type="text" required />
      </div>
      <div>
        <label htmlFor="message">Message</label>
        <textarea id="message" name="message" rows={3} required />
      </div>
      <button type="submit">Sign Guestbook</button>
    </form>
  );
}
```

### Verification: It Just Works

With JavaScript still disabled, refresh the page and submit the form again.

**Browser Behavior (JS Disabled)**:
The page performs a full refresh. After reloading, the new guestbook entry is visible at the top of the list. The form has successfully submitted.

**Browser Behavior (JS Enabled)**:
Now, re-enable JavaScript and submit the form. The behavior is identical to before: a smooth, client-side update without a full page refresh.

**What's happening under the hood?**

When you pass a Server Action directly to `<form action={...}>`, Next.js does two things:
1.  **For non-JS clients**: It renders a standard HTML `<form>` with the `action` attribute pointing to a special Next.js endpoint. The browser handles the submission as a normal form post, leading to a full page reload.
2.  **For JS clients**: It still renders the standard HTML form, but it also "hydrates" it with a client-side script. This script intercepts the `submit` event, prevents the full page reload, and performs the RPC call in the background, providing the smooth SPA-like experience.

This is the essence of progressive enhancement. The form is functional at its core using platform primitives and is then enhanced for a better user experience when JavaScript is available.

**Limitation preview**: This is great, but we've lost our loading and error states. If the submission takes time, the user gets no feedback. And if the user submits invalid data (e.g., an empty name), the form just reloads with no indication of what went wrong. We need to handle server-side validation and communicate the form's state back to the user.

## Error handling and validation

## Building Resilient Forms

A form without validation is a bug waiting to happen. We need to ensure that the data submitted by the user meets our requirements before we save it to the database. This validation must happen on the server, as client-side validation can be easily bypassed.

We also need a way to communicate the results—success or failure—back to the user.

### The Problem: Handling Invalid Data

Let's modify our Server Action to throw an error if the message is too short.

```typescript
// src/app/actions.ts (updated)

'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function addEntry(data: FormData) {
  const name = data.get('name') as string;
  const message = data.get('message') as string;

  if (!name || !message) {
    throw new Error('Name and message are required');
  }

  if (message.length < 5) {
    throw new Error('Message must be at least 5 characters long.'); // New validation
  }

  await db.addEntry({ name, message });
  revalidatePath('/');
}
```

Now, run the application and try to submit a message with fewer than 5 characters, like "hi".

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The application crashes. The user is shown the Next.js error overlay with the message "Error: Message must be at least 5 characters long." This is a terrible user experience. We should never show raw error pages for simple validation failures.

**Terminal Output (Next.js Server)**:
```bash
- error src/app/actions.ts (14:10) @ addEntry
- error Error: Message must be at least 5 characters long.
    at addEntry (./src/app/actions.ts:18:11)
    ...
```

**Let's parse this evidence**:
1.  **What the user experiences**: A full-page crash. They lose their input and are confused.
2.  **What the server reveals**: The `throw new Error(...)` call in our action resulted in an unhandled promise rejection, which Next.js treats as a fatal error for that request.
3.  **Root cause identified**: We are using exceptions (`throw`) for control flow. Errors should be reserved for unexpected server failures, not for predictable user input validation.
4.  **What we need**: A way for our Server Action to return structured data, including validation errors, that our component can use to render helpful feedback to the user without crashing the application.

### Iteration 4: Returning Structured State from Server Actions

Instead of throwing an error, a Server Action can return a JSON-serializable object. We can use this to communicate the form's state. Let's define a state shape and use a validation library like `zod` for more robust validation.

First, install zod: `npm install zod`.

**Project Structure Update**:
```
src/
└── lib/
    └── validations.ts      <- New file for our Zod schema
```

```typescript
// src/lib/validations.ts

import { z } from 'zod';

export const GuestbookSchema = z.object({
  name: z.string().min(1, { message: 'Name cannot be empty.' }),
  message: z.string().min(5, { message: 'Message must be at least 5 characters.' }),
});
```

Now, let's update our action to use this schema and return a structured state object.

```typescript
// src/app/actions.ts (updated)

'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';
import { GuestbookSchema } from '@/lib/validations';

export interface FormState {
  success: boolean;
  message: string;
  errors?: {
    name?: string[];
    message?: string[];
  };
}

export async function addEntry(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  // 1. Validate the form data
  const validatedFields = GuestbookSchema.safeParse({
    name: formData.get('name'),
    message: formData.get('message'),
  });

  // 2. If validation fails, return the errors
  if (!validatedFields.success) {
    return {
      success: false,
      message: 'Validation failed.',
      errors: validatedFields.error.flatten().fieldErrors,
    };
  }

  // 3. If validation succeeds, update the database
  try {
    await db.addEntry(validatedFields.data);
    revalidatePath('/');
    return { success: true, message: 'Entry added successfully!' };
  } catch (e) {
    return { success: false, message: 'Failed to add entry.' };
  }
}
```

Notice the function signature has changed to `(prevState, formData)`. This is the signature required by the `useFormState` hook.

### The `useFormState` Hook

React provides a hook, `useFormState`, specifically for managing form state with Server Actions. It takes the action and an initial state as arguments and returns the current state and a new action to pass to the form.

Let's create our final, robust `GuestbookForm` component.

```tsx
// src/components/GuestbookForm.tsx (final version)

'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { addEntry, FormState } from '@/app/actions';

const initialState: FormState = {
  success: false,
  message: '',
};

function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button
      type="submit"
      disabled={pending}
      className="inline-flex justify-center py-2 px-4 border border-transparent shadow-sm text-sm font-medium rounded-md text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500 disabled:opacity-50"
    >
      {pending ? 'Submitting...' : 'Sign Guestbook'}
    </button>
  );
}

export function GuestbookForm() {
  const [state, formAction] = useFormState(addEntry, initialState);

  return (
    <form action={formAction} className="mt-6 space-y-4">
      <div>
        <label htmlFor="name" className="block text-sm font-medium text-gray-700">Name</label>
        <input id="name" name="name" type="text" required className="mt-1 block w-full ..."/>
        {state.errors?.name && (
          <p className="text-sm text-red-500 mt-1">{state.errors.name.join(', ')}</p>
        )}
      </div>
      <div>
        <label htmlFor="message" className="block text-sm font-medium text-gray-700">Message</label>
        <textarea id="message" name="message" rows={3} required className="mt-1 block w-full ..."/>
        {state.errors?.message && (
          <p className="text-sm text-red-500 mt-1">{state.errors.message.join(', ')}</p>
        )}
      </div>
      
      <SubmitButton />

      {state.message && (
        <p className={state.success ? 'text-green-500' : 'text-red-500'}>
          {state.message}
        </p>
      )}
    </form>
  );
}
```

### Analysis of the Final Form

1.  **`useFormState(addEntry, initialState)`**: This hook wires up our component to the server action. `state` will hold the return value of the last action execution. `formAction` is a new function that we pass to our `<form>`.
2.  **`useFormStatus()`**: This hook provides the pending status of the form submission. It *must* be used in a component that is a child of the `<form>`. This is why we created a separate `SubmitButton` component. It allows us to disable the button during submission without any manual `useState`.
3.  **Displaying Errors**: We can now read `state.errors` and display validation messages directly below the corresponding fields.
4.  **Success/Failure Messages**: We also display a general message from `state.message`, styled differently based on the `state.success` flag.

This final version is robust, provides excellent user feedback, and maintains progressive enhancement. If JavaScript is disabled, the form will still submit, the page will reload, and the server can render the page with the validation messages included in the initial HTML.

### Common Failure Modes and Their Signatures

#### Symptom: Server Action doesn't update the UI

**Browser behavior**: You submit the form, the network request succeeds, and you can see the data in the database if you check, but the list of entries on the page doesn't update.
**Network Tab**: A `POST` request to the page URL is successful (200 OK).
**Root cause**: You forgot to call `revalidatePath('/')` or `revalidateTag(...)` at the end of your successful Server Action. Next.js doesn't know it needs to refetch the data for the page.
**Solution**: Add `revalidatePath('/')` after the database operation in your action.

#### Symptom: `useFormStatus` returns `pending: false` always

**Browser behavior**: The submit button is never disabled.
**DevTools clues**: Check your component tree.
**Root cause**: The component calling `useFormStatus` is not a descendant of the `<form>` element it's supposed to be tracking. It might be a sibling or a parent.
**Solution**: Extract the button (or any component using `useFormStatus`) into its own component and render it *inside* the `<form>`.

#### Symptom: Type error `Property 'X' does not exist on type 'FormDataEntryValue | null'`

**Terminal Output**:
```bash
error TS2339: Property 'length' does not exist on type 'string | File | null'.
```
**Root cause**: You are getting a value from `FormData` using `formData.get('fieldName')` and assuming it's a string. It could also be a `File` (for file inputs) or `null`.
**Solution**: Type-cast it or perform a type check before using string methods: `const message = formData.get('message') as string;` or `if (typeof message === 'string') { ... }`.

## Synthesis: The Complete Journey

## The Journey: From Problem to Solution

We've traveled from a disconnected client-side form to a robust, progressively enhanced solution. Let's recap the evolution.

| Iteration | Approach              | Failure Mode / Limitation                               | Technique Applied          | Result                                                              |
| :-------- | :-------------------- | :------------------------------------------------------ | :------------------------- | :------------------------------------------------------------------ |
| 0         | Client-Side Only      | No server communication; data cannot be saved.          | None                       | A non-functional UI.                                                |
| 1         | API Route + `fetch`   | Client-side boilerplate for state management.           | API Routes, `fetch`        | Functional, but verbose client code.                                |
| 2         | Basic Server Action   | Relies on JavaScript; fails without it.                 | `action={async () => {}}`  | Simpler client code, but lacks progressive enhancement.             |
| 3         | Enhanced Server Action| No feedback for loading, errors, or validation.         | `action={serverAction}`    | Works without JS, but has a poor UX for validation.                 |
| 4         | Action with `useFormState` | N/A (Final Form)                                        | `useFormState`, `zod`      | Robust, progressively enhanced form with validation and feedback. |

### Final Implementation

Here is the complete, production-ready code for our Guestbook feature, integrating all the best practices we've learned.

**Server Action**:

```typescript
// src/app/actions.ts

'use server';

import { z } from 'zod';
import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

const GuestbookSchema = z.object({
  name: z.string().min(1, { message: 'Name cannot be empty.' }),
  message: z.string().min(5, { message: 'Message must be at least 5 characters.' }),
});

export interface FormState {
  success: boolean;
  message: string;
  errors?: z.ZodIssue[];
}

export async function addEntry(prevState: FormState, formData: FormData): Promise<FormState> {
  const validatedFields = GuestbookSchema.safeParse(Object.fromEntries(formData.entries()));

  if (!validatedFields.success) {
    return {
      success: false,
      message: 'Validation failed.',
      errors: validatedFields.error.issues,
    };
  }

  try {
    await db.addEntry(validatedFields.data);
    revalidatePath('/');
    return { success: true, message: 'Entry added successfully!' };
  } catch (e) {
    return { success: false, message: 'Database error: Failed to add entry.' };
  }
}
```

**Form Component**:

```tsx
// src/components/GuestbookForm.tsx

'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { addEntry, FormState } from '@/app/actions';
import { useEffect, useRef } from 'react';

const initialState: FormState = { success: false, message: '' };

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending} className="...">
      {pending ? 'Submitting...' : 'Sign Guestbook'}
    </button>
  );
}

export function GuestbookForm() {
  const [state, formAction] = use_form_state(addEntry, initialState);
  const formRef = useRef<HTMLFormElement>(null);

  useEffect(() => {
    if (state.success) {
      formRef.current?.reset();
    }
  }, [state.success]);

  const getErrors = (fieldName: string) => 
    state.errors?.filter((issue) => issue.path[0] === fieldName).map(issue => issue.message);

  return (
    <form ref={formRef} action={formAction} className="mt-6 space-y-4">
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" name="name" type="text" required />
        {getErrors('name')?.map(error => <p key={error} className="text-sm text-red-500 mt-1">{error}</p>)}
      </div>
      <div>
        <label htmlFor="message">Message</label>
        <textarea id="message" name="message" rows={3} required />
        {getErrors('message')?.map(error => <p key={error} className="text-sm text-red-500 mt-1">{error}</p>)}
      </div>
      <SubmitButton />
      {state.message && !state.errors && (
        <p className={state.success ? 'text-green-500' : 'text-red-500'}>
          {state.message}
        </p>
      )}
    </form>
  );
}
```

### Decision Framework: API Routes vs. Server Actions

When should you reach for an API Route versus a Server Action?

| Use Case                               | Recommended Approach | Why?                                                                                                                              |
| :------------------------------------- | :------------------- | :-------------------------------------------------------------------------------------------------------------------------------- |
| **Form submissions from React components** | **Server Action**    | Drastically reduces client-side boilerplate, enables progressive enhancement, and integrates tightly with React hooks like `useFormState`. |
| **Exposing an endpoint for a third-party service (e.g., webhooks)** | **API Route**        | Provides a standard, stable REST or GraphQL endpoint that external services can call. Server Actions are not designed for this. |
| **Fetching data from a non-React client (e.g., mobile app, another backend)** | **API Route**        | API Routes are framework-agnostic. Anyone who can make an HTTP request can consume them.                                          |
| **Simple RPC-style calls from client components (e.g., clicking a "like" button)** | **Server Action**    | It's simpler than setting up a `fetch` call. You can invoke an action without a `<form>` by calling it directly in an event handler. |
| **Implementing a full REST API**       | **API Route**        | The file-based routing and explicit HTTP method handlers (`GET`, `POST`, `PUT`) map perfectly to REST principles.                     |

**General Rule of Thumb**:
*   For any **mutation (create, update, delete) triggered by your Next.js UI**, prefer **Server Actions**.
*   For any endpoint that needs to be **consumed by something other than your Next.js UI**, use **API Routes**.
