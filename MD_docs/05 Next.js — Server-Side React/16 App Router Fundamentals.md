# Chapter 16: App Router Fundamentals

## File-based routing

## The App Router: A New Mental Model

Welcome to the Next.js App Router. This is a fundamental shift from the traditional `pages` directory and introduces powerful new capabilities built on top of React Server Components. The core idea is simple but profound: **the file system is your API for routing and UI structure.**

Instead of managing a complex routing configuration file, you create folders and files within the `app` directory, and Next.js automatically maps them to URL paths. This convention-over-configuration approach simplifies routing and co-locates all files related to a specific route—pages, layouts, loading states, and error UIs—in one place.

### Phase 1: Establish the Reference Implementation

Throughout this chapter, we will build a simple **Project Dashboard** application. This will serve as our anchor example to explore the App Router's features.

Our goal is to build an application with the following routes:
1.  `/`: The main dashboard, showing a list of projects.
2.  `/projects/[id]`: A detailed view for a specific project.
3.  `/settings`: A settings page.

Let's start by creating the basic file structure. When you create a new Next.js project with `create-next-app`, you'll get an `app` directory. We'll build inside it.

**Project Structure**:
```
app/
├── favicon.ico
├── globals.css
├── layout.tsx         # Root Layout (more on this later)
└── page.tsx           # The homepage at '/'
```

### The `page.tsx` Convention

The most fundamental file in the App Router is `page.tsx`. Any folder containing a `page.tsx` file becomes a publicly accessible route. The content of `page.tsx` is the UI that will be rendered for that route segment.

Let's define our homepage.

**File: `app/page.tsx`**

```tsx
import Link from 'next/link';

// Mock data for our projects
const projects = [
  { id: 'ares-1', name: 'Ares I Rocket' },
  { id: 'orion-5', name: 'Orion V Capsule' },
  { id: 'gateway-3', name: 'Lunar Gateway' },
];

export default function DashboardPage() {
  return (
    <div>
      <h1 style={{ fontFamily: 'sans-serif', fontSize: '2rem', marginBottom: '1rem' }}>Project Dashboard</h1>
      <ul style={{ listStyle: 'none', padding: 0 }}>
        {projects.map((project) => (
          <li key={project.id} style={{ marginBottom: '0.5rem' }}>
            <Link 
              href={`/projects/${project.id}`}
              style={{ color: 'blue', textDecoration: 'underline' }}
            >
              {project.name}
            </Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

When you run your development server (`npm run dev`) and navigate to `http://localhost:3000`, you will see this component rendered. The file `app/page.tsx` maps directly to the `/` URL path.

### Static vs. Dynamic Routes

What we've just created is a **static route**. The path `/` is fixed. But what about the project detail pages, like `/projects/ares-1`? The project ID is dynamic.

To create dynamic routes, you use folders with square brackets `[]`. This tells Next.js that this part of the URL is a variable placeholder.

Let's create the dynamic route for our projects.

1.  Create a new folder named `projects` inside `app`.
2.  Inside `projects`, create another folder named `[projectId]`. The name inside the brackets, `projectId`, will be the name of the parameter we receive in our component.
3.  Inside `[projectId]`, create a `page.tsx` file.

**New Project Structure**:
```
app/
├── projects/
│   └── [projectId]/
│       └── page.tsx   # The dynamic project page
├── layout.tsx
└── page.tsx
```

Now, let's write the code for this dynamic page. Next.js will pass the dynamic segments of the URL as a `params` object to our page component.

**File: `app/projects/[projectId]/page.tsx`**

```tsx
// This component receives params because it's in a dynamic route folder.
type ProjectPageProps = {
  params: {
    projectId: string;
  };
};

// We'll add some mock data fetching logic.
// In a real app, this would be an API call.
async function getProjectDetails(id: string) {
  console.log(`Fetching data for project: ${id}`);
  // Simulate network delay
  await new Promise(resolve => setTimeout(resolve, 1000));
  
  const allProjects: { [key: string]: { name: string; description: string } } = {
    'ares-1': { name: 'Ares I Rocket', description: 'A crew-launch vehicle for the Constellation program.' },
    'orion-5': { name: 'Orion V Capsule', description: 'A reusable crewed spacecraft used in the Artemis program.' },
    'gateway-3': { name: 'Lunar Gateway', description: 'A planned small space station in lunar orbit.' },
  };

  return allProjects[id] || null;
}

export default async function ProjectPage({ params }: ProjectPageProps) {
  const project = await getProjectDetails(params.projectId);

  if (!project) {
    return <div>Project not found.</div>;
  }

  return (
    <div>
      <h1 style={{ fontFamily: 'sans-serif', fontSize: '2rem' }}>{project.name}</h1>
      <p style={{ fontFamily: 'sans-serif' }}>{project.description}</p>
    </div>
  );
}
```

Now, if you click on the "Ares I Rocket" link from the dashboard, your browser will navigate to `/projects/ares-1`. Next.js will match this URL to the `app/projects/[projectId]/page.tsx` file and render the `ProjectPage` component, passing `{ projectId: 'ares-1' }` as the `params` prop.

This file-based system is the foundation of the App Router. It's intuitive, co-located, and powerful. You've now seen how to create both static (`/`) and dynamic (`/projects/[projectId]`) routes.

## Server Components vs. Client Components

## The Two Flavors of React: Server and Client

The most significant architectural concept introduced with the App Router is the distinction between **Server Components** and **Client Components**. Understanding this is crucial for building modern, performant Next.js applications.

By default, **all components inside the `app` directory are Server Components.**

**Server Components:**
*   Run **only** on the server.
*   Their code is never sent to the browser, reducing your JavaScript bundle size.
*   Can directly access server-side resources (like databases, file systems, APIs) without an extra API call.
*   Can be `async/await` to fetch data.
*   Cannot use hooks like `useState` or `useEffect` because they are not interactive and don't run in the browser.

**Client Components:**
*   Are pre-rendered on the server (for the initial page load) and then "hydrated" to run in the browser.
*   Their code is sent to the browser, adding to the JavaScript bundle.
*   Are what you need for interactivity: `onClick` handlers, state management (`useState`, `useReducer`), and lifecycle effects (`useEffect`).
*   You opt-in to making a component a Client Component by adding the `"use client";` directive at the very top of the file.

The mental model is this: build your UI with Server Components by default for performance, and sprinkle in "islands" of interactivity with Client Components where needed.

### Iteration 1: The Inevitable Failure

Our `ProjectPage` is currently a Server Component. It fetches data directly using `async/await`, which is great. But what happens when we try to add interactivity?

Let's add a simple "Like" button to our project page. A user should be able to click it and see a count increase.

**The "Wrong" Way: Adding state to a Server Component**

Let's try to add `useState` directly to our `ProjectPage`.

**File: `app/projects/[projectId]/page.tsx` (Attempt 1)**

```tsx
// This will fail!
import { useState } from 'react';

type ProjectPageProps = {
  params: {
    projectId: string;
  };
};

async function getProjectDetails(id: string) {
  // ... (same as before)
  await new Promise(resolve => setTimeout(resolve, 1000));
  const allProjects: { [key: string]: { name: string; description: string } } = {
    'ares-1': { name: 'Ares I Rocket', description: 'A crew-launch vehicle for the Constellation program.' },
    'orion-5': { name: 'Orion V Capsule', description: 'A reusable crewed spacecraft used in the Artemis program.' },
    'gateway-3': { name: 'Lunar Gateway', description: 'A planned small space station in lunar orbit.' },
  };
  return allProjects[id] || null;
}

export default async function ProjectPage({ params }: ProjectPageProps) {
  const [likes, setLikes] = useState(0); // <-- PROBLEM
  const project = await getProjectDetails(params.projectId);

  if (!project) {
    return <div>Project not found.</div>;
  }

  return (
    <div>
      <h1 style={{ fontFamily: 'sans-serif', fontSize: '2rem' }}>{project.name}</h1>
      <p style={{ fontFamily: 'sans-serif' }}>{project.description}</p>
      <button onClick={() => setLikes(likes + 1)}>
        ❤️ {likes} Likes
      </button>
    </div>
  );
}
```

When you save this file, your Next.js development server will immediately throw an error.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
You will see a Next.js error overlay instead of your page.

**Terminal Output**:
```bash
./app/projects/[projectId]/page.tsx
ReactServerComponentsError:

You're importing a component that needs useState. It only works in a Client Component but none of its parents are marked with "use client", so they're Server Components by default.

   ,-[/Users/you/dev/project-dashboard/app/projects/[projectId]/page.tsx:23:1]
23 |   const [likes, setLikes] = useState(0); // <-- PROBLEM
   :         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
24 |   const project = await getProjectDetails(params.projectId);
25 | 
   `----

Maybe one of these should be marked as a client entry with "use client":
  ./app/projects/[projectId]/page.tsx
```

**Let's parse this evidence**:

1.  **What the user experiences**: The application crashes and shows an error page.
2.  **What the console reveals**: The error message is incredibly clear. It identifies the exact line of code (`const [likes, setLikes] = useState(0);`) and explains the problem: `useState` only works in a Client Component.
3.  **Root cause identified**: We are trying to use a client-side hook (`useState`) inside a Server Component.
4.  **Why the current approach can't solve this**: Server Components are fundamentally non-interactive. They run once on the server to generate HTML and have no mechanism to re-render in the browser based on user events.
5.  **What we need**: We need to isolate the interactive part of our UI (the button) into its own Client Component, leaving the data-fetching part as a Server Component.

### The Solution: Creating an "Island of Interactivity"

The correct pattern is to keep `ProjectPage` as a Server Component to handle the data fetching, and create a new, separate `LikeButton` component that is explicitly marked as a Client Component.

First, let's create the new component.

**File: `app/projects/[projectId]/LikeButton.tsx`**

```tsx
"use client"; // <-- This is the magic directive!

import { useState } from 'react';

export default function LikeButton() {
  const [likes, setLikes] = useState(0);

  return (
    <button 
      onClick={() => setLikes(likes + 1)}
      style={{ marginTop: '1rem', padding: '8px 16px', cursor: 'pointer' }}
    >
      ❤️ {likes} Likes
    </button>
  );
}
```

Now, we revert our `ProjectPage` to be a pure Server Component again and import our new `LikeButton`.

**File: `app/projects/[projectId]/page.tsx` (Corrected)**

```tsx
import LikeButton from './LikeButton'; // <-- Import the Client Component

type ProjectPageProps = {
  params: {
    projectId: string;
  };
};

async function getProjectDetails(id: string) {
  // ... (same as before)
  await new Promise(resolve => setTimeout(resolve, 1000));
  const allProjects: { [key: string]: { name: string; description: string } } = {
    'ares-1': { name: 'Ares I Rocket', description: 'A crew-launch vehicle for the Constellation program.' },
    'orion-5': { name: 'Orion V Capsule', description: 'A reusable crewed spacecraft used in the Artemis program.' },
    'gateway-3': { name: 'Lunar Gateway', description: 'A planned small space station in lunar orbit.' },
  };
  return allProjects[id] || null;
}

// This is a Server Component again!
export default async function ProjectPage({ params }: ProjectPageProps) {
  const project = await getProjectDetails(params.projectId);

  if (!project) {
    return <div>Project not found.</div>;
  }

  return (
    <div>
      <h1 style={{ fontFamily: 'sans-serif', fontSize: '2rem' }}>{project.name}</h1>
      <p style={{ fontFamily: 'sans-serif' }}>{project.description}</p>
      
      {/* We can seamlessly use a Client Component inside a Server Component */}
      <LikeButton />
    </div>
  );
}
```

### Verification

Now, when you navigate to a project page:
1.  The `ProjectPage` component runs on the server.
2.  It fetches the data for the specific project.
3.  It renders the static HTML for the title and description.
4.  It includes a placeholder for the `LikeButton` component.
5.  The server sends the initial HTML to the browser.
6.  The browser then downloads the JavaScript for `LikeButton.tsx`.
7.  React "hydrates" the `LikeButton`, making it interactive.

When you click the button, only the `LikeButton` component re-renders in your browser. The parent `ProjectPage` is unaffected. This is the power of the App Router's architecture: you get the performance benefits of server rendering with the surgical interactivity of client-side React.

## Layouts and templates

## Shared UI: Layouts and Templates

Our application is functional, but it lacks a consistent structure. The dashboard and project pages are completely separate. In any real application, you'll want a shared header, navigation, or footer across multiple pages. This is the job of the `layout.tsx` file.

### The Problem: Repetitive UI

The naive way to create a shared header would be to create a `<Header />` component and manually import it into every single `page.tsx` file.

**The "Wrong" Way (for demonstration):**
```tsx
// app/page.tsx
import Header from '../components/Header';
export default function DashboardPage() {
  return (
    <div>
      <Header />
      <h1>Project Dashboard</h1>
      {/* ... */}
    </div>
  );
}

// app/projects/[projectId]/page.tsx
import Header from '../../../components/Header';
export default async function ProjectPage({ params }) {
  // ...
  return (
    <div>
      <Header />
      <h1>{project.name}</h1>
      {/* ... */}
    </div>
  );
}
```

This approach has two major flaws:
1.  **Repetition:** You have to remember to add `<Header />` to every new page.
2.  **Performance:** When you navigate between the dashboard and a project page, the `<Header />` component will be unmounted and remounted. This means any state within the header (like a search query or a user dropdown) would be lost, and the entire component re-renders unnecessarily.

### The Solution: The `layout.tsx` Convention

Next.js solves this with another special file: `layout.tsx`. A layout is a UI component that wraps child layouts or pages. When you navigate between pages that share the same layout, the layout component itself **does not re-render**. Its state is preserved.

#### The Root Layout

Every Next.js App Router project must have a root layout, located at `app/layout.tsx`. This layout wraps your entire application. It's where you should define the `<html>` and `<body>` tags.

Let's create a proper root layout with a shared header.

**File: `app/layout.tsx`**

```tsx
import Link from 'next/link';
import './globals.css'; // Assuming you have some basic styles

// The RootLayout receives a `children` prop, which will be the page component.
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <header style={{ 
          padding: '1rem', 
          borderBottom: '1px solid #ccc', 
          marginBottom: '2rem' 
        }}>
          <nav>
            <Link href="/" style={{ marginRight: '1rem', fontWeight: 'bold' }}>
              Dashboard
            </Link>
            {/* We can add more links here later */}
          </nav>
        </header>
        <main style={{ padding: '0 1rem' }}>
          {children}
        </main>
      </body>
    </html>
  );
}
```

With this file in place, you can now remove the manual `<Header />` imports from your page files. The `children` prop will be automatically populated by Next.js with the active page component. Now, both the dashboard (`/`) and the project pages (`/projects/...`) will be rendered inside this layout, giving them a consistent header.

When you navigate between these pages, you'll notice in the browser that only the content inside `<main>` changes. The header remains static, which is exactly what we want.

### Nested Layouts

Layouts can also be nested to create more specific shared UI for certain sections of your app. For example, perhaps all pages under `/projects` should have a special sub-navigation bar.

We can achieve this by adding a `layout.tsx` file inside the `app/projects/` directory.

**New Project Structure**:
```
app/
├── projects/
│   ├── [projectId]/
│   │   ├── LikeButton.tsx
│   │   └── page.tsx
│   └── layout.tsx       # <-- Nested layout for the /projects route
├── layout.tsx           # <-- Root layout
└── page.tsx
```

This new layout will wrap all pages within the `projects` segment, including `projects/[projectId]`.

**File: `app/projects/layout.tsx`**

```tsx
import Link from 'next/link';

// This layout will be nested inside the RootLayout
export default function ProjectsLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      <nav style={{ 
        padding: '0.5rem', 
        background: '#f0f0f0', 
        marginBottom: '1rem',
        borderRadius: '4px'
      }}>
        {/* This is a conceptual link, we don't have this page yet */}
        <Link href="/projects/new">New Project</Link>
      </nav>
      {children}
    </div>
  );
}
```

Now, when you visit a project page like `/projects/ares-1`, the component tree will look like this:

1.  `RootLayout` (from `app/layout.tsx`)
2.  `ProjectsLayout` (from `app/projects/layout.tsx`)
3.  `ProjectPage` (from `app/projects/[projectId]/page.tsx`)

The `children` of `RootLayout` will be `ProjectsLayout`, and the `children` of `ProjectsLayout` will be `ProjectPage`. This powerful nesting allows you to build complex, hierarchical UI structures that map directly to your URL paths.

### A Note on `template.tsx`

There is another, less common special file called `template.tsx`. It's similar to a layout in that it wraps a page, but with one key difference: **a new instance of the `template` component is mounted on every navigation.**

This means state is *not* preserved. This can be useful for features that rely on `useEffect` to run on every page navigation (e.g., for analytics tracking) or for implementing enter/exit animations with CSS. In most cases, `layout.tsx` is what you want, but it's good to know `template.tsx` exists for these specific use cases.

## Loading and error states

## Handling UI States: Loading and Errors

Our application works, but the user experience is still rough around the edges. What happens when data fetching is slow? Or what if it fails entirely? The App Router provides elegant, file-based conventions for handling these states using React Suspense and Error Boundaries under the hood.

### The Problem: Slow Data Fetching

In our `ProjectPage`, we simulated a 1-second network delay.
```typescript
// app/projects/[projectId]/page.tsx
await new Promise(resolve => setTimeout(resolve, 1000));
```
When you navigate from the dashboard to a project page, the browser hangs for a full second before the new page appears. The UI feels unresponsive. This is a poor user experience, especially on slow connections.

### The Solution: Instant Loading UI with `loading.tsx`

Next.js has a magical solution for this: the `loading.tsx` file. By creating a file named `loading.tsx` alongside a `page.tsx`, you tell Next.js to automatically show this component as a fallback while the data for the page is being fetched.

Let's add one for our project pages.

**File: `app/projects/[projectId]/loading.tsx`**

```tsx
export default function ProjectLoading() {
  // You can add any UI inside Loading, including a Skeleton.
  return (
    <div>
      <h1 style={{ 
        height: '2rem', 
        width: '300px', 
        background: '#eee', 
        borderRadius: '4px',
        marginBottom: '1rem'
      }}></h1>
      <p style={{ 
        height: '1rem', 
        width: '500px', 
        background: '#eee', 
        borderRadius: '4px' 
      }}></p>
    </div>
  );
}
```

That's it. You don't need to import this file anywhere. By its very existence, Next.js will now do the following:
1.  When a user navigates to `/projects/ares-1`, Next.js will **immediately** render the `ProjectLoading` component.
2.  The shared layouts (`RootLayout` and `ProjectsLayout`) will also render instantly.
3.  In the background, Next.js will start fetching the data in `ProjectPage`.
4.  Once the data fetching is complete, it will swap out `ProjectLoading` with the fully rendered `ProjectPage`.

This is achieved using React Suspense. Next.js automatically wraps your `page.tsx` in a `<Suspense>` boundary, using your `loading.tsx` as the `fallback`. This enables **Streaming Server Rendering**, where the static parts of the page are sent to the browser first, followed by the dynamic content as it becomes ready.

### The Problem: Data Fetching Fails

What happens if we try to visit a project that doesn't exist, like `/projects/invalid-id`? Our `getProjectDetails` function will return `null`, and our component currently handles this gracefully.

```tsx
// app/projects/[projectId]/page.tsx
if (!project) {
  return <div>Project not found.</div>;
}
```

But what if the data source itself throws an error? Let's simulate this.

**File: `app/projects/[projectId]/page.tsx` (Simulating an error)**

```tsx
// ...
async function getProjectDetails(id: string) {
  console.log(`Fetching data for project: ${id}`);
  await new Promise(resolve => setTimeout(resolve, 1000));

  // NEW: Simulate a failure for a specific ID
  if (id === 'orion-5') {
    throw new Error("Failed to fetch data for Orion V Capsule. The API is down!");
  }
  
  const allProjects = { /* ... */ };
  return allProjects[id] || null;
}
// ...
```

### Diagnostic Analysis: Reading the Failure

Now, if you navigate to `/projects/orion-5`:

**Browser Behavior**:
In development, you'll see the Next.js error overlay with the message "Error: Failed to fetch data...". In production, this would crash and show a generic 500 error page, taking your whole application down for that user.

**Terminal Output**:
```bash
- error ./app/projects/[projectId]/page.tsx (30:10) @ getProjectDetails
- error Error: Failed to fetch data for Orion V Capsule. The API is down!
    at getProjectDetails (./app/projects/[projectId]/page.tsx:20:11)
    ...
```

This is not ideal. A failure in one part of the UI shouldn't break the entire page. The header and navigation should still be visible and interactive.

### The Solution: Graceful Error Handling with `error.tsx`

Similar to `loading.tsx`, Next.js provides an `error.tsx` convention to handle runtime errors. This file defines a UI boundary for any errors that occur in a route segment and its children.

**Important:** An `error.tsx` component **must be a Client Component** (`"use client"`). This is because it's designed to be interactive, often including a "Try again" button to re-attempt the render.

Let's create one for our project route.

**File: `app/projects/[projectId]/error.tsx`**

```tsx
"use client"; // Error components must be Client Components

import { useEffect } from 'react';

type ErrorProps = {
  error: Error;
  reset: () => void; // A function to re-render the segment
};

export default function ProjectError({ error, reset }: ErrorProps) {
  useEffect(() => {
    // Log the error to an error reporting service
    console.error(error);
  }, [error]);

  return (
    <div style={{ 
      padding: '2rem', 
      border: '1px solid red', 
      borderRadius: '8px', 
      background: '#fff5f5' 
    }}>
      <h2>Something went wrong!</h2>
      <p style={{ color: 'red' }}>{error.message}</p>
      <button
        onClick={
          // Attempt to recover by trying to re-render the segment
          () => reset()
        }
        style={{ marginTop: '1rem' }}
      >
        Try again
      </button>
    </div>
  );
}
```

Under the hood, Next.js wraps your `page.tsx` in a React Error Boundary and uses your `error.tsx` file as the fallback UI.

Now, when you navigate to `/projects/orion-5`:
1.  The `getProjectDetails` function throws an error.
2.  Next.js catches this error.
3.  It renders the `ProjectError` component in place of the `ProjectPage` component.
4.  Crucially, the `RootLayout` and `ProjectsLayout` are **unaffected** and remain visible and interactive.

The error is contained. The user sees a helpful message and has an action they can take (`reset()`) to try and recover, all without losing the context of the rest of the application.

### The Journey: From Problem to Solution

Let's synthesize what we've built. We started with simple files and progressively added special, conventional files to handle more complex UI requirements.

| Iteration | Problem                               | File Convention Applied          | Result                                                              |
| --------- | ------------------------------------- | -------------------------------- | ------------------------------------------------------------------- |
| 0         | How to create pages?                  | `page.tsx`                       | Basic static and dynamic routes are working.                        |
| 1         | How to add interactivity?             | `"use client";`                  | Interactive logic isolated in Client Components, pages remain fast. |
| 2         | How to create shared UI?              | `layout.tsx`                     | Consistent UI across pages without re-rendering on navigation.      |
| 3         | UI hangs during data loading.         | `loading.tsx`                    | Instant loading state (skeleton UI) shown while data fetches.       |
| 4         | An error crashes the entire page.     | `error.tsx`                      | Errors are isolated, showing a fallback UI without breaking layouts.|

### Final Implementation

Here is the final file structure of our robust Project Dashboard segment:

**Final Project Structure**:
```
app/
├── projects/
│   └── [projectId]/
│       ├── error.tsx        # Handles runtime errors
│       ├── LikeButton.tsx   # Client Component for interactivity
│       ├── loading.tsx      # Loading UI (Suspense fallback)
│       └── page.tsx         # The main page component (Server Component)
├── layout.tsx
└── page.tsx
```

This structure is a perfect example of the App Router's philosophy: co-locating all files related to a route, using file conventions to define UI for different states, and defaulting to server-first for performance while allowing surgical client-side interactivity.
