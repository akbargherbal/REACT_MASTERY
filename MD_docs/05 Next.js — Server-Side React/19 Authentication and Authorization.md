# Chapter 19: Authentication and Authorization

## NextAuth.js (Auth.js) setup

## The Unprotected Application: A Security Failure by Default

Authentication isn't a feature you add at the end; it's a foundational concern. To understand its importance, we'll start with a common scenario: a simple blog application that is completely unprotected. This application will be our **anchor example** for the entire chapter.

Our goal is to build a blog where only authenticated administrators can create new posts.

### Phase 1: Establish the Reference Implementation

First, let's set up the initial, insecure version of our application.

**Project Structure**:
```
src/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                  # Public homepage
│   └── admin/
│       └── create-post/
│           └── page.tsx          # The page that SHOULD be private
└── components/
    └── Header.tsx                # A simple site header
```

Here is the code for our key files.

**The "Create Post" Page (Problematic Version)**:
This page is currently accessible to anyone who knows the URL.

```tsx
// src/app/admin/create-post/page.tsx

export default function CreatePostPage() {
  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold mb-4">Create New Post</h1>
      <form className="flex flex-col gap-4">
        <div>
          <label htmlFor="title" className="block mb-1">Title</label>
          <input type="text" id="title" name="title" className="w-full p-2 border rounded" />
        </div>
        <div>
          <label htmlFor="content" className="block mb-1">Content</label>
          <textarea id="content" name="content" rows={10} className="w-full p-2 border rounded"></textarea>
        </div>
        <button type="submit" className="bg-blue-500 text-white p-2 rounded self-start">
          Submit Post
        </button>
      </form>
    </main>
  );
}
```

**The Homepage**:
This page links to the supposedly "admin" area.

```tsx
// src/app/page.tsx
import Link from 'next/link';

export default function HomePage() {
  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold">Welcome to the Blog</h1>
      <p>This is the public homepage. Everyone can see this.</p>
    </main>
  );
}
```

**The Header**:
A shared header component.

```tsx
// src/components/Header.tsx
import Link from 'next/link';

export default function Header() {
  return (
    <header className="bg-gray-100 p-4 border-b">
      <nav className="flex justify-between">
        <Link href="/" className="font-bold">My Blog</Link>
        {/* We will add auth controls here later */}
        <Link href="/admin/create-post" className="text-blue-600 hover:underline">
          Create Post (Admin)
        </Link>
      </nav>
    </header>
  );
}
```

**The Root Layout**:
We'll add our header to the main layout.

```tsx
// src/app/layout.tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import Header from "@/components/Header";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "Auth.js Demo",
  description: "A demo of Next.js authentication",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <Header />
        {children}
      </body>
    </html>
  );
}
```

### Diagnostic Analysis: Reading the Failure

Let's run this application and diagnose its fundamental flaw.

**Browser Behavior**:
A user clicks the "Create Post (Admin)" link. They are immediately taken to the `/admin/create-post` page and can see the form to create a new post. There is no login prompt, no access check, nothing.

**Network Tab Analysis**:
- Request pattern: A simple GET request to `/admin/create-post`.
- Response codes: 200 OK.
- Cookies: No session or authentication-related cookies are sent or received.

**Let's parse this evidence**:

1.  **What the user experiences**: The application has no concept of "admin" vs. "public" areas. All routes are public.
    -   **Expected**: Accessing `/admin/create-post` should require a login.
    -   **Actual**: Anyone can access `/admin/create-post`.

2.  **What the network reveals**: The server happily serves the page to any browser that asks for it. There is no authentication mechanism in place.

3.  **Root cause identified**: The application lacks an authentication system to identify users and a mechanism to protect routes based on that identity.

4.  **Why the current approach can't solve this**: Standard Next.js routing is based on the file system and is public by default. It has no built-in knowledge of user sessions or permissions.

5.  **What we need**: A robust, centralized authentication library that integrates with Next.js to manage user sessions and provide tools for protecting pages. This is the problem that Auth.js (formerly NextAuth.js) is designed to solve.

### Iteration 1: Installing and Configuring Auth.js

Let's introduce Auth.js to our project to add a basic authentication layer. We'll use the GitHub provider for a simple OAuth login.

First, install the necessary package.

```bash
npm install next-auth@beta
```

> **Note**: We are using `next-auth@beta` which is the latest version, commonly referred to as Auth.js v5, designed for the Next.js App Router.

Next, we need to set up environment variables for our authentication provider and a secret key for signing session cookies.

**Create a `.env.local` file** in the root of your project.

```text
# .env.local

# Generate a secret with: openssl rand -base64 32
AUTH_SECRET="your-super-secret-value-here"

# GitHub OAuth App credentials
AUTH_GITHUB_ID="your-github-client-id"
AUTH_GITHUB_SECRET="your-github-client-secret"
```

You can get your GitHub credentials by creating a new OAuth App in your GitHub developer settings. The "Authorization callback URL" should be `http://localhost:3000/api/auth/callback/github`.

Now, we create the core of our authentication logic: the Auth.js configuration and API route handler.

**Project Structure Change**:
```
src/
├── app/
│   ├── api/
│   │   └── auth/
│   │       └── [...nextauth]/
│   │           └── route.ts      # ← New Auth.js API route
...
├── auth.ts                       # ← New Auth.js config file
...
```

**The Auth.js Configuration**:
This file defines our authentication strategies (providers).

```typescript
// src/auth.ts
import NextAuth from "next-auth"
import GitHub from "next-auth/providers/github"

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [GitHub],
})
```

**The Auth.js API Route**:
This file exposes the `GET` and `POST` handlers from our configuration as a catch-all API route. This single route handles sign-in, sign-out, callbacks, and session management.

```typescript
// src/app/api/auth/[...nextauth]/route.ts
export { handlers as GET, handlers as POST } from "@/auth"
```

With these two files, we now have a fully functional set of authentication API endpoints. However, our application's UI doesn't know about them yet. We need to provide a way for users to sign in and out.

Let's create a component to handle this.

```tsx
// src/components/AuthButtons.tsx
"use client";

import { signIn, signOut, useSession } from "next-auth/react";

export default function AuthButtons() {
  const { data: session } = useSession();

  if (session) {
    return (
      <div className="flex items-center gap-4">
        <p>
          Signed in as {session.user?.email}
        </p>
        <button onClick={() => signOut()} className="bg-red-500 text-white px-3 py-1 rounded">
          Sign Out
        </button>
      </div>
    );
  }

  return (
    <button onClick={() => signIn("github")} className="bg-blue-500 text-white px-3 py-1 rounded">
      Sign In with GitHub
    </button>
  );
}
```

This component uses the `useSession` hook, which requires a client-side context provider. We must wrap our application in `SessionProvider`.

**Updating the Root Layout**:

```tsx
// src/app/layout.tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import Header from "@/components/Header";
import { SessionProvider } from "next-auth/react"; // ← Import

const inter = Inter({ subsets: ["latin"] });

export const metadata = { /* ... */ };

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <SessionProvider> {/* ← Wrap children */}
          <Header />
          {children}
        </SessionProvider>
      </body>
    </html>
  );
}
```

Finally, let's replace the hardcoded link in our `Header` with the new `AuthButtons` component.

**Updating the Header**:

```tsx
// src/components/Header.tsx
import Link from 'next/link';
import AuthButtons from './AuthButtons'; // ← Import

export default function Header() {
  return (
    <header className="bg-gray-100 p-4 border-b">
      <nav className="flex justify-between items-center">
        <div className="flex items-center gap-4">
          <Link href="/" className="font-bold">My Blog</Link>
          <Link href="/admin/create-post" className="text-blue-600 hover:underline">
            Create Post
          </Link>
        </div>
        <AuthButtons /> {/* ← Use the component */}
      </nav>
    </header>
  );
}
```

### Verification

Let's run the app now.
- The header now shows a "Sign In with GitHub" button.
- Clicking it redirects you to GitHub to authorize the application.
- After authorizing, you are redirected back to the homepage.
- The header now displays "Signed in as your.email@example.com" and a "Sign Out" button.

**Expected vs. Actual Improvement**:
- **Expected**: A way for users to establish an identity within the application.
- **Actual**: We have a complete login/logout flow. The application can now distinguish between authenticated and anonymous users.

**Limitation Preview**:
This is a huge step forward, but we've only solved half the problem. We can *identify* users, but we aren't *using* that identity to control access. Anyone, logged in or not, can still navigate directly to `/admin/create-post` and see the form. Our next step is to use the session data to protect content.

## Session management

## Accessing the User Session

Our application now has a concept of a "logged-in user," but this information is currently confined to the `AuthButtons` component. To make our application truly auth-aware, we need to access the user's session data in different contexts:
1.  **Client Components**: To conditionally render UI elements (like we did with `AuthButtons`).
2.  **Server Components**: To fetch user-specific data or render content on the server based on the user's identity.
3.  **API Routes / Route Handlers**: To authorize API requests.

Auth.js provides different methods for each context.

### Iteration 2: Making the Application Session-Aware

**Current State Recap**: Users can log in and out, but the rest of the application is oblivious to their authentication status. The `/admin/create-post` page is still wide open.

**Current Limitation**: We can't personalize content or protect pages because we aren't checking for a valid session outside of our `Header`.

**New Scenario Introduction**: What if we want to greet the user by name on the homepage? And what if the "Create Post" page should only be visible to logged-in users?

### Accessing the Session in Server Components

Let's start by personalizing the homepage, which is a Server Component. We can use the `auth()` function exported from our `src/auth.ts` file.

**Before**: The homepage is static.
```tsx
// src/app/page.tsx (Old version)
import Link from 'next/link';

export default function HomePage() {
  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold">Welcome to the Blog</h1>
      <p>This is the public homepage. Everyone can see this.</p>
    </main>
  );
}
```

**After**: The homepage greets the logged-in user.

```tsx
// src/app/page.tsx (New version)
import { auth } from "@/auth"; // ← Import auth

export default async function HomePage() {
  const session = await auth(); // ← Get session on the server

  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold">Welcome to the Blog</h1>
      {session?.user ? (
        <p>Hello, {session.user.name}! You are signed in.</p>
      ) : (
        <p>This is the public homepage. Please sign in to create a post.</p>
      )}
    </main>
  );
}
```

Now, when a logged-in user visits the homepage, they see a personalized greeting. An anonymous user sees the original message. This happens entirely on the server, resulting in a fast, non-interactive initial page load.

### Accessing the Session in Client Components

We've already seen this in action with our `AuthButtons` component. The `useSession` hook is the primary way to access session data in Client Components. It's provided by the `SessionProvider` we added in `layout.tsx`.

Let's try to "protect" our `create-post` page using this client-side technique. This approach is flawed, but it's a crucial step in understanding *why* we need server-side protection.

**Attempting Client-Side Protection**:
We'll convert the `CreatePostPage` to a Client Component and use `useSession` to check for a user.

```tsx
// src/app/admin/create-post/page.tsx (Client-side protection attempt)
"use client";

import { useSession } from "next-auth/react";
import { useRouter } from "next/navigation";
import { useEffect } from "react";

export default function CreatePostPage() {
  const { data: session, status } = useSession();
  const router = useRouter();

  useEffect(() => {
    // If the session is loading, do nothing yet.
    if (status === 'loading') return;

    // If there is no session, redirect to the homepage.
    if (!session) {
      router.push('/');
    }
  }, [session, status, router]);

  // While loading, show a message.
  if (status === 'loading') {
    return <p className="p-8">Loading session...</p>;
  }

  // If there's no session after loading, the redirect is in progress.
  // Render null or a message to avoid flashing the protected content.
  if (!session) {
    return <p className="p-8">Redirecting...</p>;
  }

  // If we have a session, render the protected content.
  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold mb-4">Create New Post</h1>
      <p className="mb-4">Welcome, {session.user?.name}!</p>
      <form className="flex flex-col gap-4">
        {/* Form fields... */}
      </form>
    </main>
  );
}
```

### Diagnostic Analysis: The Failure of Client-Side Protection

Let's analyze what happens when a logged-out user tries to access `/admin/create-post`.

**Browser Behavior**:
The user sees a "Loading session..." message for a brief moment, which is then replaced by "Redirecting...". The page content might flash on the screen for a split second before the JavaScript kicks in and redirects them to the homepage. This is known as a "flash of unprotected content."

**Network Tab Analysis**:
1.  The browser sends a GET request for the `/admin/create-post` page.
2.  The server, which doesn't know about the user's session for this page, sends back the full HTML and JavaScript for the `CreatePostPage` component (including the form!).
3.  The browser renders this page.
4.  The client-side JavaScript runs. The `useSession` hook makes a request to `/api/auth/session` to get the session data.
5.  The session API returns that there is no active session.
6.  The `useEffect` hook runs, sees there is no session, and triggers `router.push('/')`.
7.  The browser navigates to the homepage.

**Let's parse this evidence**:

1.  **What the user experiences**: A flicker of the protected page before being redirected. It feels slow and insecure.

2.  **What the network reveals**: The key failure is that the protected content (the HTML for the form) is sent to the browser *before* the authentication check is complete.

3.  **Root cause identified**: The protection logic runs on the client, but the content is delivered from the server. The server has already sent the sensitive data before the client has a chance to stop it.

4.  **Why this approach can't solve this**: Client-side rendering and redirection for security is fundamentally flawed. It's a UX problem (the flicker) and a security risk (sensitive data is sent to the browser, even if briefly).

5.  **What we need**: A way to perform the authentication check on the server *before* any of the page's HTML is rendered or sent to the client. This is the job of Next.js Middleware.

## Protected routes and middleware

## Server-Side Protection with Middleware

We've seen that client-side redirects are insufficient for protecting routes. The check must happen on the server before the request is handed over to the page component. In Next.js, the perfect place for this is Middleware.

Middleware is a function that runs before a request is completed. Based on the incoming request, you can rewrite, redirect, or modify headers before passing the request along to be rendered.

### Iteration 3: Implementing Middleware for Route Protection

**Current State Recap**: We can access session data, but our attempt to protect the `/admin/create-post` page on the client-side resulted in a "flash of unprotected content."

**Current Limitation**: Our security check happens too late in the request lifecycle.

**New Scenario Introduction**: How can we ensure that a request from a logged-out user to `/admin/create-post` is *never* allowed to reach the page component, and is instead redirected to a login page instantly?

First, let's simplify our `CreatePostPage` back to a Server Component. It no longer needs to worry about redirecting; it can assume that if it renders, the user is authenticated.

```tsx
// src/app/admin/create-post/page.tsx (Simplified Server Component)
import { auth } from "@/auth";

export default async function CreatePostPage() {
  const session = await auth();

  // We can be reasonably sure session exists because of middleware,
  // but it's good practice to handle the edge case.
  if (!session?.user) {
    return <p className="p-8">Access Denied</p>;
  }

  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold mb-4">Create New Post</h1>
      <p className="mb-4">Welcome, {session.user.name}!</p>
      <form className="flex flex-col gap-4">
        <div>
          <label htmlFor="title" className="block mb-1">Title</label>
          <input type="text" id="title" name="title" className="w-full p-2 border rounded" />
        </div>
        <div>
          <label htmlFor="content" className="block mb-1">Content</label>
          <textarea id="content" name="content" rows={10} className="w-full p-2 border rounded"></textarea>
        </div>
        <button type="submit" className="bg-blue-500 text-white p-2 rounded self-start">
          Submit Post
        </button>
      </form>
    </main>
  );
}
```

Now, let's create the middleware file. This file must be placed at the root of your project (or inside `src/`).

**Project Structure Change**:
```
src/
├── middleware.ts                 # ← New middleware file
...
```

Auth.js v5 provides a convenient way to integrate with Next.js middleware. We can simply export the `auth` function from our `auth.ts` file as the default export in `middleware.ts`.

**The Middleware Implementation**:

```typescript
// src/middleware.ts
export { auth as default } from "@/auth"
```

This is a great start, but by default, this protects *all* routes in your application, which is not what we want. We need to tell the middleware which routes to protect and which to leave public. We do this by adding a `matcher` configuration to our `auth.ts` file.

Let's update our Auth.js config to include authorization logic.

```typescript
// src/auth.ts (Updated with authorization logic)
import NextAuth from "next-auth"
import GitHub from "next-auth/providers/github"
import type { NextAuthConfig } from "next-auth"

export const config = {
  theme: {
    logo: "https://next-auth.js.org/img/logo/logo-sm.png",
  },
  providers: [GitHub],
  callbacks: {
    authorized({ request, auth }) {
      const { pathname } = request.nextUrl
      // Protect any route under /admin
      if (pathname.startsWith("/admin")) {
        // Returns true if the user is logged in, false otherwise.
        return !!auth
      }
      // All other routes are public
      return true
    },
  },
} satisfies NextAuthConfig

export const { handlers, auth, signIn, signOut } = NextAuth(config)
```

The `authorized` callback is the heart of our middleware's logic.
1.  It receives the `request` and the `auth` object (the session).
2.  We check if the request's `pathname` starts with `/admin`.
3.  If it does, we return `!!auth`. This expression converts the `auth` object (which is either the session object or `null`) into a boolean. If the user is logged in, it returns `true` (access granted). If not, it returns `false` (access denied).
4.  If the route is not under `/admin`, we return `true`, making it public.

When `authorized` returns `false`, Auth.js will automatically redirect the user to the sign-in page.

### Verification

Let's test the new behavior as a logged-out user:
1.  Navigate to the homepage (`/`). It loads normally.
2.  Click the "Create Post" link, or manually type `http://localhost:3000/admin/create-post` into the address bar.

**Browser Behavior**:
You are instantly redirected to the Auth.js default sign-in page, which prompts you to "Sign in with GitHub". There is no flicker, no flash of unprotected content.

**Network Tab Analysis**:
1.  The browser sends a GET request for `/admin/create-post`.
2.  The Next.js server runs the middleware *before* looking for the page component.
3.  The `authorized` callback runs, finds no session (`auth` is `null`), and returns `false`.
4.  The middleware intercepts the request and returns a `307 Temporary Redirect` response, with the `Location` header pointing to `/api/auth/signin?callbackUrl=%2Fadmin%2Fcreate-post`.
5.  The browser follows the redirect to the sign-in page.

**Expected vs. Actual Improvement**:
- **Expected**: A secure way to prevent unauthorized access to a route.
- **Actual**: We have implemented robust, server-side route protection. The protected page's code is never even executed, let alone sent to an unauthorized user's browser.

**Limitation Preview**:
Our application can now distinguish between anonymous users and logged-in users. But what if we have different *types* of logged-in users? For example, "subscribers" who can read content and "admins" who can write content. Currently, *any* user who logs in via GitHub can access the `/admin` area. We need a more granular level of control: Role-Based Access Control (RBAC).

## Role-based access control

## Implementing Role-Based Access Control (RBAC)

Authentication answers the question, "Who are you?". Authorization answers the question, "What are you allowed to do?". So far, we've only implemented authentication. Now, we'll add authorization by assigning roles to users.

For our blog, we'll define two roles:
-   `reader`: A standard logged-in user.
-   `admin`: A user who can create posts.

### Iteration 4: Adding Roles to the Session

**Current State Recap**: Any user who successfully logs in can access the `/admin` routes.

**Current Limitation**: Our authorization logic is binary: you're either logged in or you're not. It doesn't account for different permission levels among authenticated users.

**New Scenario Introduction**: How can we ensure that only a specific user (e.g., the one whose email matches a predefined admin email) can access `/admin/create-post`, while all other logged-in users are denied access?

To implement this, we need to inject custom data—our `role` field—into the Auth.js session token. We can do this using the `jwt` and `session` callbacks in our `auth.ts` configuration.

1.  **`jwt` callback**: This is called whenever a JSON Web Token is created or updated. We can use it to add the user's role to the token itself.
2.  **`session` callback**: This is called whenever a session is accessed. We use it to take the data from the token (like our custom role) and make it available on the session object that our application code uses.

Let's update `auth.ts` to include these callbacks. For this example, we'll hardcode a single email address as the admin. In a real application, this would come from a database.

```typescript
// src/auth.ts (Updated with RBAC)
import NextAuth from "next-auth"
import GitHub from "next-auth/providers/github"
import type { NextAuthConfig } from "next-auth"

// Define the admin email in an environment variable for security
const ADMIN_EMAIL = process.env.ADMIN_EMAIL;

export const config = {
  theme: { /* ... */ },
  providers: [GitHub],
  callbacks: {
    // This callback is used to customize the JWT.
    async jwt({ token, user }) {
      // On initial sign-in, the `user` object is available.
      if (user) {
        // Check if the signed-in user is the admin.
        token.role = user.email === ADMIN_EMAIL ? "admin" : "reader";
      }
      return token;
    },

    // This callback is used to customize the session object.
    async session({ session, token }) {
      // Add the role from the token to the session object.
      if (session?.user) {
        session.user.role = token.role as string;
      }
      return session;
    },
    
    authorized({ request, auth }) {
      const { pathname } = request.nextUrl
      if (pathname.startsWith("/admin")) {
        // Check for both a session AND the admin role.
        return auth?.user?.role === "admin";
      }
      return true
    },
  },
} satisfies NextAuthConfig

export const { handlers, auth, signIn, signOut } = NextAuth(config)
```

And add the admin email to your `.env.local` file:

```text
# .env.local
...
ADMIN_EMAIL="your-admin-email@example.com"
```

> **Type Safety Note**: The default `session.user` object doesn't have a `role` property. To make TypeScript aware of our custom property, you can extend the `next-auth` types using module augmentation. Create a file `types/next-auth.d.ts`:

```typescript
// types/next-auth.d.ts
import 'next-auth';

declare module 'next-auth' {
  interface Session {
    user: {
      role?: string;
    } & DefaultSession['user'];
  }
  
  interface User {
    role?: string;
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    role?: string;
  }
}
```

### Diagnostic Analysis: Testing the Role-Based Failure

Let's demonstrate the failure this new system prevents.
1.  Log out of the application.
2.  Log in with a GitHub account whose email is **not** the `ADMIN_EMAIL`. You are now a `reader`.
3.  Try to navigate to `/admin/create-post`.

**Browser Behavior**:
You are logged in, but when you try to access the admin page, you are redirected back to the sign-in page with an "Access Denied" error message. The system correctly identifies that while you are authenticated, you are not authorized.

**Let's parse this evidence**:
1.  **What the user experiences**: A clear denial of access, even though they are logged in.
    -   **Expected**: Only admins should see the create post page.
    -   **Actual**: The middleware correctly blocks non-admin users.
2.  **How it works**:
    -   When you logged in, the `jwt` callback ran and assigned `token.role = "reader"`.
    -   When you requested `/admin/create-post`, the middleware ran.
    -   The `authorized` callback was executed. It found a session (`auth` was not null).
    -   However, the check `auth?.user?.role === "admin"` failed because `auth.user.role` was `"reader"`.
    -   The callback returned `false`, triggering the redirect to the sign-in page with an error.

### Conditionally Rendering UI Based on Role

We can also use this new `role` property to conditionally render UI elements. For example, we could hide the "Create Post" link from users who are not admins.

**Updating the Header**:
This requires converting `Header` to a Server Component to access the session.

```tsx
// src/components/Header.tsx (Updated for RBAC)
import Link from 'next/link';
import AuthButtons from './AuthButtons';
import { auth } from '@/auth'; // ← Import auth

export default async function Header() {
  const session = await auth(); // ← Get session on the server
  const isAdmin = session?.user?.role === 'admin';

  return (
    <header className="bg-gray-100 p-4 border-b">
      <nav className="flex justify-between items-center">
        <div className="flex items-center gap-4">
          <Link href="/" className="font-bold">My Blog</Link>
          {isAdmin && ( // ← Conditionally render the link
            <Link href="/admin/create-post" className="text-blue-600 hover:underline">
              Create Post
            </Link>
          )}
        </div>
        <AuthButtons />
      </nav>
    </header>
  );
}
```

Now, a user with the `reader` role won't even see the link to the admin section, providing a cleaner user experience on top of the robust server-side security.

### Common Failure Modes and Their Signatures

#### Symptom: Custom session properties (like `role`) are `undefined`.

**Browser behavior**:
UI that depends on the role doesn't render, or server-side checks fail.

**Console pattern**:
```
TypeError: Cannot read properties of undefined (reading 'role')
```

**DevTools clues**:
- In React DevTools, inspecting the `useSession` hook shows a `session` object, but the `user` object is missing the custom field.

**Root cause**: The `jwt` and/or `session` callbacks in `auth.ts` are missing or incorrect. You must pass the custom data from the `token` to the `session` object in the `session` callback.
**Solution**: Ensure both `jwt` and `session` callbacks are correctly implemented to persist the custom data.

#### Symptom: Infinite redirect loop after logging in.

**Browser behavior**:
The page continuously reloads between the page you're trying to access and the login page.

**Network Tab clues**:
- A rapid sequence of 307 redirects between, for example, `/admin` and `/api/auth/signin`.

**Root cause**: The middleware logic is flawed. A common mistake is protecting a route that is part of the login flow itself, or a logic error in the `authorized` callback that denies access even to valid users.
**Solution**: Carefully review the `authorized` callback logic. Use `console.log(pathname, auth)` inside the callback to trace what's happening on each request. Ensure you are not protecting public pages like the homepage if that's where you redirect after login.

#### Symptom: `Error: NEXT_AUTH_SECRET is not set` or `AUTH_SECRET is not set`

**Terminal Output**:
```bash
[next-auth][error][NO_SECRET]
https://next-auth.js.org/errors#no_secret
```

**Root cause**: The `AUTH_SECRET` (or `NEXTAUTH_SECRET` for older versions) environment variable is missing from `.env.local` or is not accessible by the server process. This is critical for production builds.
**Solution**: Generate a strong secret (`openssl rand -base64 32`) and add it to your `.env.local` and your production environment variables.

## Synthesis - The Complete Journey

## The Journey: From Unprotected to Role-Based Security

We have progressively transformed our application from a completely open public site to a secure application with granular, role-based access control. Each step solved a critical flaw in the previous iteration.

| Iteration | Failure Mode                               | Technique Applied                               | Result                                           | Security Posture |
| :-------- | :----------------------------------------- | :---------------------------------------------- | :----------------------------------------------- | :--------------- |
| 0         | All routes are public by default.          | None                                            | Anyone can access the admin page.                | Insecure         |
| 1         | No user identity.                          | Install and configure Auth.js with a provider.  | Users can log in and out.                        | Authenticated    |
| 2         | Protected content flashes on screen.       | Client-side `useSession` check and redirect.    | Content is hidden after a delay, but still sent. | Flawed           |
| 3         | Client-side checks are insecure.           | Next.js Middleware with `authorized` callback.  | Unauthorized requests are blocked on the server. | Secure           |
| 4         | All logged-in users have the same access.  | RBAC via `jwt` and `session` callbacks.         | Access is granted based on user role.            | Granular         |

### Final Implementation

Here is the complete, production-ready code for our authentication system, incorporating all the improvements.

**Final `src/auth.ts`**:

```typescript
// src/auth.ts
import NextAuth from "next-auth"
import GitHub from "next-auth/providers/github"
import type { NextAuthConfig } from "next-auth"

const ADMIN_EMAIL = process.env.ADMIN_EMAIL;

if (!ADMIN_EMAIL) {
  throw new Error("ADMIN_EMAIL environment variable is not set");
}

export const config = {
  theme: {
    logo: "https://next-auth.js.org/img/logo/logo-sm.png",
  },
  providers: [
    GitHub({
      clientId: process.env.AUTH_GITHUB_ID,
      clientSecret: process.env.AUTH_GITHUB_SECRET,
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = user.email === ADMIN_EMAIL ? "admin" : "reader";
      }
      return token;
    },

    async session({ session, token }) {
      if (session?.user) {
        session.user.role = token.role as string;
      }
      return session;
    },
    
    authorized({ request, auth }) {
      const { pathname } = request.nextUrl;
      if (pathname.startsWith("/admin")) {
        return auth?.user?.role === "admin";
      }
      // All other routes are public and do not require authentication.
      return true;
    },
  },
} satisfies NextAuthConfig

export const { handlers, auth, signIn, signOut } = NextAuth(config)
```

**Final `src/middleware.ts`**:

```typescript
// src/middleware.ts
export { auth as default } from "@/auth"

// Optionally, you can use a matcher to specify which routes the middleware should run on.
// This is often more performant than running it on every request.
// export const config = {
//   matcher: ["/admin/:path*"],
// };
```

### Decision Framework: Authorization Strategies

| Strategy                               | When to Use                                                                                             | Pros                                                              | Cons                                                               |
| :------------------------------------- | :------------------------------------------------------------------------------------------------------ | :---------------------------------------------------------------- | :----------------------------------------------------------------- |
| **Middleware (`authorized` callback)** | For protecting entire route segments (`/admin`, `/dashboard`). The primary line of defense.               | Runs first, highly secure, blocks requests on the server, central logic. | Less granular than per-component checks.                               |
| **Server Component (`await auth()`)**  | To fetch user-specific data or make authorization decisions within a page that is already protected.      | Integrates seamlessly with server-side data fetching.             | Runs after middleware; should not be the *only* line of defense.   |
| **Client Component (`useSession`)**    | For purely cosmetic UI changes (e.g., showing/hiding a "Profile" link). **Never for security.**           | Responsive, provides loading states.                              | Insecure for protecting content, causes layout shifts/flickering.  |

### Lessons Learned

1.  **Security is Server-First**: True security for web applications must be enforced on the server. Client-side checks are for user experience, not for protection.
2.  **Middleware is the Gatekeeper**: Next.js Middleware is the ideal tool for protecting routes, as it runs at the edge before any page-specific logic is executed.
3.  **Separate Authentication from Authorization**: Auth.js provides the tools for both. Use providers for authentication (`who you are`) and callbacks/middleware for authorization (`what you can do`).
4.  **The Session is Your Source of Truth**: By enriching the session token with custom data like roles, you create a single, reliable source of truth for authorization decisions throughout your application.
