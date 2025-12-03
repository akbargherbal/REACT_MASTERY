# Chapter 19: Authentication and Authorization

## NextAuth.js (Auth.js) setup

## The Failure: Insecure Client-Only Auth Checks

Let's start with what most developers build first—and why it's fundamentally broken.

### Reference Implementation: E-commerce Admin Dashboard

We're building an admin dashboard for our e-commerce product catalog. Admins need to:

- View all products (including unpublished ones)
- Edit product details
- Manage inventory
- View customer orders

Here's the naive approach that seems to work:

```tsx
// app/admin/page.tsx
'use client';

import { useState, useEffect } from 'react';
import { useRouter } from 'next/navigation';

export default function AdminDashboard() {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [products, setProducts] = useState([]);
  const router = useRouter();

  useEffect(() => {
    // Check if user is logged in
    const token = localStorage.getItem('authToken');
    
    if (!token) {
      router.push('/login');
      return;
    }

    setIsAuthenticated(true);

    // Fetch admin data
    fetch('/api/admin/products', {
      headers: {
        'Authorization': `Bearer ${token}`
      }
    })
      .then(res => res.json())
      .then(data => setProducts(data));
  }, [router]);

  if (!isAuthenticated) {
    return <div>Checking authentication...</div>;
  }

  return (
    <div>
      <h1>Admin Dashboard</h1>
      <div>
        {products.map(product => (
          <div key={product.id}>
            <h2>{product.name}</h2>
            <p>Price: ${product.price}</p>
            <p>Stock: {product.inventory}</p>
            <button>Edit</button>
          </div>
        ))}
      </div>
    </div>
  );
}
```

This code runs. It redirects unauthenticated users. It fetches admin data. Ship it, right?

**Wrong. This is security theater, not security.**

### Diagnostic Analysis: Reading the Failure

Let's see what an attacker sees when they open DevTools:

**Browser Behavior**:
1. Page loads and shows "Checking authentication..." for a split second
2. Then redirects to `/login`
3. But if you're fast enough (or disable JavaScript), you can see the admin UI

**Browser Console Output**:
```
GET /api/admin/products 401 Unauthorized
```

**React DevTools Evidence**:
- Component tree shows: `AdminDashboard` → rendered
- State: `isAuthenticated: false`, `products: []`
- The component fully mounts before the redirect happens

**Network Tab Analysis**:
- Request to `/api/admin/products` fires immediately
- Response: 401 Unauthorized
- But the request was made—the API endpoint was discovered

**Let's parse this evidence**:

1. **What the user experiences**: Brief flash of admin UI, then redirect

2. **What the console reveals**: The API endpoint `/api/admin/products` is exposed in the client-side code

3. **What DevTools shows**: 
   - The entire admin component renders before auth check completes
   - All product data structure is visible in the component code
   - localStorage token is visible in Application tab

4. **Root cause identified**: Authentication happens in the browser, after the page loads

5. **Why the current approach can't solve this**: Client-side code is public code. Any "protection" that happens in the browser can be bypassed by:
   - Disabling JavaScript
   - Modifying localStorage
   - Editing the component code in DevTools
   - Directly calling API endpoints with tools like curl

6. **What we need**: Authentication that happens on the server, before any protected content is sent to the browser

### The Fundamental Problem: Client-Side Auth is Not Auth

Here's what an attacker can do with 30 seconds and DevTools:

**Attack 1: Disable JavaScript**
```bash
# In Chrome DevTools: Settings → Debugger → Disable JavaScript
# Now visit /admin
# Result: Full admin UI renders (no redirect happens)
```

**Attack 2: Modify localStorage**
```javascript
// In browser console:
localStorage.setItem('authToken', 'fake-token-12345');
// Refresh page
// Result: Passes client-side check, makes API request
```

**Attack 3: Direct API Access**
```bash
# The API endpoint is visible in the source code
curl https://yoursite.com/api/admin/products \
  -H "Authorization: Bearer fake-token"

# If the API doesn't validate properly, you get data
```

**Attack 4: View Source**
```html
<!-- View page source -->
<!-- All component code is visible, including: -->
<!-- - API endpoints -->
<!-- - Data structures -->
<!-- - Business logic -->
```

### What We Actually Need

Authentication must happen in three places, in this order:

1. **Server-side route protection**: Check auth before rendering the page
2. **API route protection**: Validate tokens on every API request
3. **Client-side UX**: Show appropriate UI based on auth state (but never rely on it for security)

Client-side checks are for user experience, not security. They prevent confusion, not attacks.

## NextAuth.js: Server-Side Auth for Next.js

NextAuth.js (now called Auth.js) solves this by:

1. Managing sessions on the server
2. Providing middleware to protect routes before they render
3. Handling OAuth providers (Google, GitHub, etc.)
4. Encrypting session tokens
5. Giving you hooks to check auth state in components

### Installation and Setup

First, install the dependencies:

```bash
npm install next-auth@beta
```

**Note**: We're using `next-auth@beta` because it's the version compatible with Next.js 13+ App Router. The stable version only works with Pages Router.

### Project Structure

Here's how we'll organize our auth setup:

```plaintext
src/
├── app/
│   ├── api/
│   │   └── auth/
│   │       └── [...nextauth]/
│   │           └── route.ts          ← Auth API routes
│   ├── admin/
│   │   ├── page.tsx                  ← Protected admin page
│   │   └── products/
│   │       └── [id]/
│   │           └── page.tsx          ← Protected product editor
│   ├── login/
│   │   └── page.tsx                  ← Login page
│   └── layout.tsx
├── lib/
│   └── auth.ts                       ← Auth configuration
└── middleware.ts                     ← Route protection
```

### Core Auth Configuration

Create the auth configuration file:

```typescript
// lib/auth.ts
import NextAuth from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import { compare } from 'bcryptjs';

// This would come from your database
// For now, we'll use a mock
async function getUserFromDatabase(email: string) {
  // In production, this queries your database
  // Example: await db.user.findUnique({ where: { email } })
  
  // Mock user for demonstration
  if (email === 'admin@example.com') {
    return {
      id: '1',
      email: 'admin@example.com',
      name: 'Admin User',
      role: 'admin',
      // This is bcrypt hash of 'password123'
      passwordHash: '$2a$10$rXQvvXvXvXvXvXvXvXvXvXvXvXvXvXvXvXvXvXvXvXvXvXvXvXvXv'
    };
  }
  
  return null;
}

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    CredentialsProvider({
      name: 'Credentials',
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" }
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          return null;
        }

        const user = await getUserFromDatabase(credentials.email as string);
        
        if (!user) {
          return null;
        }

        const isValidPassword = await compare(
          credentials.password as string,
          user.passwordHash
        );

        if (!isValidPassword) {
          return null;
        }

        // Return user object (without password)
        return {
          id: user.id,
          email: user.email,
          name: user.name,
          role: user.role
        };
      }
    })
  ],
  callbacks: {
    async jwt({ token, user }) {
      // Add user info to JWT token
      if (user) {
        token.id = user.id;
        token.role = user.role;
      }
      return token;
    },
    async session({ session, token }) {
      // Add user info to session
      if (session.user) {
        session.user.id = token.id as string;
        session.user.role = token.role as string;
      }
      return session;
    }
  },
  pages: {
    signIn: '/login',
  },
  session: {
    strategy: 'jwt',
  },
});
```

### Understanding the Configuration

Let's break down what each part does:

**Providers**: Define how users authenticate
- `CredentialsProvider`: Username/password login
- Could also use `GoogleProvider`, `GitHubProvider`, etc.

**authorize function**: Validates credentials
- Queries database for user
- Compares password hash
- Returns user object if valid, null if not

**Callbacks**: Customize JWT and session data
- `jwt`: Runs when JWT is created/updated—add custom data here
- `session`: Runs when session is accessed—shape the session object

**pages**: Custom auth pages
- `signIn`: Where to redirect for login

**session.strategy**: How sessions are stored
- `jwt`: Stateless, encrypted token in cookie (recommended)
- `database`: Store sessions in database (more control, more complexity)

### API Route Handler

Create the catch-all route for NextAuth:

```typescript
// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/lib/auth';

export const { GET, POST } = handlers;
```

This single file handles all auth endpoints:
- `GET /api/auth/signin` - Sign in page
- `POST /api/auth/signin` - Sign in submission
- `GET /api/auth/signout` - Sign out
- `GET /api/auth/session` - Get current session
- And more...

### TypeScript Types

Extend NextAuth types to include our custom fields:

```typescript
// types/next-auth.d.ts
import 'next-auth';

declare module 'next-auth' {
  interface User {
    role?: string;
  }
  
  interface Session {
    user: {
      id: string;
      email: string;
      name: string;
      role: string;
    };
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    id?: string;
    role?: string;
  }
}
```

### Login Page

Create a login form that uses NextAuth:

```tsx
// app/login/page.tsx
'use client';

import { signIn } from 'next-auth/react';
import { useState } from 'react';
import { useRouter } from 'next/navigation';

export default function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const router = useRouter();

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError('');
    setIsLoading(true);

    try {
      const result = await signIn('credentials', {
        email,
        password,
        redirect: false,
      });

      if (result?.error) {
        setError('Invalid email or password');
        setIsLoading(false);
        return;
      }

      // Success - redirect to admin
      router.push('/admin');
      router.refresh(); // Refresh to update server components
    } catch (err) {
      setError('An error occurred. Please try again.');
      setIsLoading(false);
    }
  }

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="max-w-md w-full space-y-8 p-8 bg-white rounded-lg shadow">
        <div>
          <h2 className="text-3xl font-bold text-center">Admin Login</h2>
        </div>
        
        <form onSubmit={handleSubmit} className="space-y-6">
          {error && (
            <div className="bg-red-50 text-red-600 p-3 rounded">
              {error}
            </div>
          )}
          
          <div>
            <label htmlFor="email" className="block text-sm font-medium">
              Email
            </label>
            <input
              id="email"
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              required
              className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md"
            />
          </div>
          
          <div>
            <label htmlFor="password" className="block text-sm font-medium">
              Password
            </label>
            <input
              id="password"
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              required
              className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md"
            />
          </div>
          
          <button
            type="submit"
            disabled={isLoading}
            className="w-full py-2 px-4 bg-blue-600 text-white rounded-md hover:bg-blue-700 disabled:opacity-50"
          >
            {isLoading ? 'Signing in...' : 'Sign In'}
          </button>
        </form>
        
        <div className="text-sm text-gray-600 text-center">
          Demo credentials: admin@example.com / password123
        </div>
      </div>
    </div>
  );
}
```

### Testing the Setup

Start your dev server and test:

```bash
npm run dev
```

**Browser Console Output** (successful login):
```
POST /api/auth/callback/credentials 200 OK
GET /api/auth/session 200 OK
```

**Network Tab Analysis**:
- `POST /api/auth/callback/credentials`: Login request
  - Request body: `{ email, password }` (encrypted in transit via HTTPS)
  - Response: Sets `next-auth.session-token` cookie
- `GET /api/auth/session`: Fetch session data
  - Response: `{ user: { id, email, name, role } }`

**Application Tab** (Chrome DevTools):
- Cookies: `next-auth.session-token` is set
  - Value: Encrypted JWT (not readable in DevTools)
  - HttpOnly: Yes (JavaScript cannot access it)
  - Secure: Yes (only sent over HTTPS)
  - SameSite: Lax (CSRF protection)

This is already more secure than our localStorage approach:
- Token is HttpOnly (can't be stolen via XSS)
- Token is encrypted (can't be tampered with)
- Token is validated on the server

But we still haven't protected our admin routes. Let's fix that next.

## Session management

## Accessing Session Data

Now that users can log in, we need to access their session data in our components and API routes.

### In Server Components

Server Components can directly access the session:

```tsx
// app/admin/page.tsx
import { auth } from '@/lib/auth';
import { redirect } from 'next/navigation';

export default async function AdminDashboard() {
  const session = await auth();

  if (!session) {
    redirect('/login');
  }

  // Fetch admin data server-side
  const products = await fetch('http://localhost:3000/api/admin/products', {
    headers: {
      // Pass session info to API
      'Cookie': `next-auth.session-token=${session.user.id}`
    }
  }).then(res => res.json());

  return (
    <div>
      <h1>Admin Dashboard</h1>
      <p>Welcome, {session.user.name}</p>
      <p>Role: {session.user.role}</p>
      
      <div className="grid gap-4 mt-8">
        {products.map((product: any) => (
          <div key={product.id} className="border p-4 rounded">
            <h2 className="text-xl font-bold">{product.name}</h2>
            <p>Price: ${product.price}</p>
            <p>Stock: {product.inventory}</p>
            <a 
              href={`/admin/products/${product.id}`}
              className="text-blue-600 hover:underline"
            >
              Edit
            </a>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**What changed from our broken version**:

1. **Server Component**: No `'use client'` directive—this runs on the server
2. **Direct session access**: `await auth()` gets session server-side
3. **Redirect before render**: If no session, redirect happens on the server
4. **No flash of content**: User never sees the admin UI if not authenticated

**Browser Behavior**:
- Unauthenticated user visits `/admin`
- Server checks session, finds none
- Server responds with 307 redirect to `/login`
- Browser never receives admin HTML

**View Source**:
```html
<!-- Unauthenticated user sees: -->
<!DOCTYPE html>
<html>
<head>
  <meta http-equiv="refresh" content="0;url=/login">
</head>
</html>
```

No admin code. No API endpoints. No data structures. Just a redirect.

### In Client Components

For interactive components, use the `useSession` hook:

```tsx
// app/admin/products/[id]/page.tsx
'use client';

import { useSession } from 'next-auth/react';
import { useRouter } from 'next/navigation';
import { useEffect, useState } from 'react';

export default function ProductEditor({ params }: { params: { id: string } }) {
  const { data: session, status } = useSession();
  const router = useRouter();
  const [product, setProduct] = useState<any>(null);

  useEffect(() => {
    if (status === 'unauthenticated') {
      router.push('/login');
    }
  }, [status, router]);

  useEffect(() => {
    if (status === 'authenticated') {
      fetch(`/api/admin/products/${params.id}`)
        .then(res => res.json())
        .then(data => setProduct(data));
    }
  }, [status, params.id]);

  if (status === 'loading') {
    return <div>Loading...</div>;
  }

  if (!session) {
    return null;
  }

  if (!product) {
    return <div>Loading product...</div>;
  }

  return (
    <div>
      <h1>Edit Product</h1>
      <form>
        <div>
          <label>Name</label>
          <input 
            type="text" 
            value={product.name}
            onChange={(e) => setProduct({ ...product, name: e.target.value })}
          />
        </div>
        {/* More form fields */}
      </form>
    </div>
  );
}
```

**Session status values**:
- `loading`: Session is being fetched
- `authenticated`: User is logged in
- `unauthenticated`: User is not logged in

**Important**: This client-side check is still for UX only. The real protection comes from:
1. Middleware (which we'll add next)
2. API route validation (which we'll implement)

### Session Provider

To use `useSession`, wrap your app in a session provider:

```tsx
// app/layout.tsx
import { SessionProvider } from 'next-auth/react';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <SessionProvider>
          {children}
        </SessionProvider>
      </body>
    </html>
  );
}
```

### In API Routes

Protect API endpoints by checking the session:

```typescript
// app/api/admin/products/route.ts
import { auth } from '@/lib/auth';
import { NextResponse } from 'next/server';

export async function GET() {
  const session = await auth();

  if (!session) {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 401 }
    );
  }

  // Check role
  if (session.user.role !== 'admin') {
    return NextResponse.json(
      { error: 'Forbidden' },
      { status: 403 }
    );
  }

  // Fetch products from database
  const products = [
    { id: 1, name: 'Product 1', price: 29.99, inventory: 100 },
    { id: 2, name: 'Product 2', price: 49.99, inventory: 50 },
  ];

  return NextResponse.json(products);
}

export async function POST(request: Request) {
  const session = await auth();

  if (!session || session.user.role !== 'admin') {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 401 }
    );
  }

  const body = await request.json();

  // Validate and create product
  // In production: await db.product.create({ data: body })

  return NextResponse.json({ success: true });
}
```

**Testing API Protection**:

Try calling the API without authentication:

```bash
curl http://localhost:3000/api/admin/products
```

**Response**:
```json
{
  "error": "Unauthorized"
}
```

**Status code**: 401

Now try with a valid session (after logging in through the browser):

```bash
curl http://localhost:3000/api/admin/products \
  -H "Cookie: next-auth.session-token=YOUR_TOKEN_HERE"
```

**Response**:
```json
[
  { "id": 1, "name": "Product 1", "price": 29.99, "inventory": 100 },
  { "id": 2, "name": "Product 2", "price": 49.99, "inventory": 50 }
]
```

**Status code**: 200

### Session Refresh and Expiration

By default, NextAuth sessions expire after 30 days. Configure this:

```typescript
// lib/auth.ts
export const { handlers, auth, signIn, signOut } = NextAuth({
  // ... other config
  session: {
    strategy: 'jwt',
    maxAge: 30 * 24 * 60 * 60, // 30 days
    updateAge: 24 * 60 * 60, // 24 hours
  },
});
```

**maxAge**: How long until session expires
**updateAge**: How often to refresh the session token

When a session is about to expire, NextAuth automatically refreshes it on the next request.

### Sign Out

Implement sign out functionality:

```tsx
// components/SignOutButton.tsx
'use client';

import { signOut } from 'next-auth/react';

export function SignOutButton() {
  return (
    <button
      onClick={() => signOut({ callbackUrl: '/login' })}
      className="px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700"
    >
      Sign Out
    </button>
  );
}
```

Add it to your admin layout:

```tsx
// app/admin/layout.tsx
import { auth } from '@/lib/auth';
import { SignOutButton } from '@/components/SignOutButton';
import { redirect } from 'next/navigation';

export default async function AdminLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const session = await auth();

  if (!session) {
    redirect('/login');
  }

  return (
    <div>
      <header className="bg-gray-800 text-white p-4">
        <div className="container mx-auto flex justify-between items-center">
          <h1 className="text-xl font-bold">Admin Dashboard</h1>
          <div className="flex items-center gap-4">
            <span>{session.user.name}</span>
            <SignOutButton />
          </div>
        </div>
      </header>
      <main className="container mx-auto p-4">
        {children}
      </main>
    </div>
  );
}
```

**Browser Console Output** (sign out):
```
POST /api/auth/signout 200 OK
```

**Network Tab Analysis**:
- `POST /api/auth/signout`: Sign out request
  - Response: Clears `next-auth.session-token` cookie
  - Redirect: 307 to `/login`

**Application Tab**:
- Cookies: `next-auth.session-token` is deleted

### The Failure: Session Hijacking

Even with HttpOnly cookies, sessions can still be stolen if:

1. **No HTTPS**: Cookies sent over HTTP can be intercepted
2. **XSS vulnerability**: Malicious script can make authenticated requests
3. **CSRF attack**: Attacker tricks user into making requests

**Diagnostic Analysis**:

**Attack scenario**: User visits malicious site while logged in

**Malicious site code**:
```html
<img src="https://yoursite.com/api/admin/products/delete?id=1" />
```

**Browser behavior**:
- Browser automatically includes cookies with the request
- API endpoint receives authenticated request
- Product gets deleted

**Network Tab**:
- `GET /api/admin/products/delete?id=1` with valid session cookie
- Status: 200 OK
- Result: Product deleted

**Root cause**: Browser automatically sends cookies with every request to the domain, even from other sites.

**What we need**: CSRF protection to verify requests originate from our site.

NextAuth includes CSRF protection by default:
- Every form submission includes a CSRF token
- API routes validate the token
- Cross-origin requests are rejected

But we still need one more layer: middleware to protect routes before they even render.

## Protected routes and middleware

## The Failure: Protecting Every Route Manually

Right now, every protected page needs this code:

```tsx
const session = await auth();
if (!session) {
  redirect('/login');
}
```

**Problems with this approach**:

1. **Easy to forget**: One missed check = security hole
2. **Repetitive**: Same code in every file
3. **Runs too late**: Page component starts executing before check
4. **Not DRY**: Violates "Don't Repeat Yourself"

**What we need**: A single place to protect all admin routes.

## Next.js Middleware: The Gatekeeper

Middleware runs before any page renders. It's the perfect place for auth checks.

Create a middleware file at the root of your project:

```typescript
// middleware.ts
import { auth } from '@/lib/auth';
import { NextResponse } from 'next/server';

export default auth((req) => {
  const isLoggedIn = !!req.auth;
  const isOnAdminPage = req.nextUrl.pathname.startsWith('/admin');

  if (isOnAdminPage && !isLoggedIn) {
    return NextResponse.redirect(new URL('/login', req.url));
  }

  return NextResponse.next();
});

export const config = {
  matcher: ['/admin/:path*'],
};
```

### Understanding Middleware

**How it works**:

1. User requests `/admin/products`
2. Middleware runs before the page component
3. Checks if user is authenticated
4. If not, redirects to `/login`
5. If yes, allows request to continue

**matcher**: Defines which routes the middleware applies to
- `/admin/:path*`: All routes under `/admin`
- Can use arrays: `['/admin/:path*', '/dashboard/:path*']`
- Can use regex: `/admin/(.*)`

**req.auth**: The session object (provided by NextAuth)
- `null` if not authenticated
- User object if authenticated

**NextResponse.redirect**: Server-side redirect
- Happens before page renders
- User never sees protected content

### Testing Middleware Protection

**Test 1: Unauthenticated access**

Visit `/admin` without logging in:

**Browser Behavior**:
- Immediately redirects to `/login`
- No flash of admin content
- URL changes to `/login`

**Network Tab**:
```
GET /admin 307 Temporary Redirect
Location: /login
```

**View Source**:
```html
<!-- No admin HTML sent to browser -->
```

**Test 2: Authenticated access**

Log in, then visit `/admin`:

**Browser Behavior**:
- Page loads normally
- Admin content displays

**Network Tab**:
```
GET /admin 200 OK
```

**Test 3: Direct API access**

Try to bypass middleware by calling API directly:

```bash
curl http://localhost:3000/api/admin/products
```

**Response**:
```json
{
  "error": "Unauthorized"
}
```

**Why**: API routes still check session independently. Middleware protects pages, API routes protect themselves.

### Advanced Middleware Patterns

#### Pattern 1: Role-Based Route Protection

Protect different routes for different roles:

```typescript
// middleware.ts
import { auth } from '@/lib/auth';
import { NextResponse } from 'next/server';

export default auth((req) => {
  const session = req.auth;
  const path = req.nextUrl.pathname;

  // Public routes - allow everyone
  if (path.startsWith('/login') || path === '/') {
    return NextResponse.next();
  }

  // Protected routes - require authentication
  if (!session) {
    return NextResponse.redirect(new URL('/login', req.url));
  }

  // Admin routes - require admin role
  if (path.startsWith('/admin')) {
    if (session.user.role !== 'admin') {
      return NextResponse.redirect(new URL('/unauthorized', req.url));
    }
  }

  // Manager routes - require manager or admin role
  if (path.startsWith('/manager')) {
    if (!['admin', 'manager'].includes(session.user.role)) {
      return NextResponse.redirect(new URL('/unauthorized', req.url));
    }
  }

  return NextResponse.next();
});

export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
};
```

**matcher explanation**:
- `(?!api|_next/static|_next/image|favicon.ico)`: Negative lookahead—exclude these paths
- `.*`: Match everything else
- Result: Middleware runs on all pages except API routes and static files

#### Pattern 2: Redirect After Login

Remember where user was trying to go:

```typescript
// middleware.ts
import { auth } from '@/lib/auth';
import { NextResponse } from 'next/server';

export default auth((req) => {
  const session = req.auth;
  const path = req.nextUrl.pathname;

  if (path.startsWith('/admin') && !session) {
    // Save the original URL
    const loginUrl = new URL('/login', req.url);
    loginUrl.searchParams.set('callbackUrl', path);
    return NextResponse.redirect(loginUrl);
  }

  return NextResponse.next();
});
```

Update login page to use callback URL:

```tsx
// app/login/page.tsx
'use client';

import { signIn } from 'next-auth/react';
import { useSearchParams } from 'next/navigation';

export default function LoginPage() {
  const searchParams = useSearchParams();
  const callbackUrl = searchParams.get('callbackUrl') || '/admin';

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    
    const result = await signIn('credentials', {
      email,
      password,
      callbackUrl, // Redirect here after login
    });
  }

  // ... rest of component
}
```

**User experience**:

1. User visits `/admin/products/123` (not logged in)
2. Middleware redirects to `/login?callbackUrl=/admin/products/123`
3. User logs in
4. Redirected to `/admin/products/123` (original destination)

#### Pattern 3: API Route Protection in Middleware

Protect API routes too:

```typescript
// middleware.ts
import { auth } from '@/lib/auth';
import { NextResponse } from 'next/server';

export default auth((req) => {
  const session = req.auth;
  const path = req.nextUrl.pathname;

  // Protect admin API routes
  if (path.startsWith('/api/admin')) {
    if (!session) {
      return NextResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    }

    if (session.user.role !== 'admin') {
      return NextResponse.json(
        { error: 'Forbidden' },
        { status: 403 }
      );
    }
  }

  // Protect admin pages
  if (path.startsWith('/admin')) {
    if (!session) {
      return NextResponse.redirect(new URL('/login', req.url));
    }

    if (session.user.role !== 'admin') {
      return NextResponse.redirect(new URL('/unauthorized', req.url));
    }
  }

  return NextResponse.next();
});

export const config = {
  matcher: ['/admin/:path*', '/api/admin/:path*'],
};
```

**Now both pages and APIs are protected at the middleware level.**

### The Failure: Middleware Doesn't Run Everywhere

**Problem**: Middleware doesn't run on:
- Static files (`/images/logo.png`)
- API routes in some configurations
- Server Actions

**Diagnostic Analysis**:

Try to access a protected Server Action without middleware:

```typescript
// app/actions.ts
'use server';

export async function deleteProduct(id: string) {
  // No auth check!
  await db.product.delete({ where: { id } });
  return { success: true };
}
```

**Attack**:

```typescript
// Attacker's code
fetch('/api/actions', {
  method: 'POST',
  body: JSON.stringify({
    action: 'deleteProduct',
    args: ['product-123']
  })
});
```

**Result**: Product deleted without authentication.

**Solution**: Always check auth in Server Actions:

```typescript
// app/actions.ts
'use server';

import { auth } from '@/lib/auth';

export async function deleteProduct(id: string) {
  const session = await auth();

  if (!session || session.user.role !== 'admin') {
    throw new Error('Unauthorized');
  }

  await db.product.delete({ where: { id } });
  return { success: true };
}
```

**Rule**: Middleware is the first line of defense, but every protected operation must validate auth independently.

### Unauthorized Page

Create a page for unauthorized access:

```tsx
// app/unauthorized/page.tsx
import Link from 'next/link';

export default function UnauthorizedPage() {
  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="text-center">
        <h1 className="text-4xl font-bold text-gray-900 mb-4">
          403 - Forbidden
        </h1>
        <p className="text-gray-600 mb-8">
          You don't have permission to access this page.
        </p>
        <Link 
          href="/"
          className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
        >
          Go Home
        </Link>
      </div>
    </div>
  );
}
```

### Middleware Performance Considerations

Middleware runs on every request. Keep it fast:

**Good**:
- Check session (already in memory)
- Simple role checks
- Path-based routing decisions

**Bad**:
- Database queries
- External API calls
- Complex computations

**Example of what NOT to do**:

```typescript
// ❌ BAD - Don't do this
export default auth(async (req) => {
  // This runs on EVERY request
  const user = await db.user.findUnique({
    where: { id: req.auth?.user.id }
  });
  
  const permissions = await db.permission.findMany({
    where: { userId: user.id }
  });
  
  // ... complex permission logic
});
```

**Why it's bad**: Database queries on every request kill performance.

**Better approach**: Store role in session, check permissions in the actual route:

```typescript
// ✅ GOOD - Middleware stays fast
export default auth((req) => {
  // Quick check using session data
  if (req.nextUrl.pathname.startsWith('/admin')) {
    if (req.auth?.user.role !== 'admin') {
      return NextResponse.redirect(new URL('/unauthorized', req.url));
    }
  }
  return NextResponse.next();
});

// ✅ GOOD - Detailed checks in the route
// app/admin/products/[id]/page.tsx
export default async function ProductPage({ params }: { params: { id: string } }) {
  const session = await auth();
  
  // Now we can do expensive checks
  const hasPermission = await checkProductPermission(
    session.user.id,
    params.id
  );
  
  if (!hasPermission) {
    redirect('/unauthorized');
  }
  
  // ... render page
}
```

## Role-based access control

## Iteration 4: Protected Product Management

Let's build a complete role-based access control system for our e-commerce admin.

### Requirements

We need three user roles:

1. **Admin**: Full access—can create, edit, delete products
2. **Manager**: Can edit products, view orders, but cannot delete
3. **Viewer**: Read-only access to products and orders

### Database Schema

First, let's define our user and permission structure:

```typescript
// prisma/schema.prisma
model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String
  passwordHash  String
  role          Role      @default(VIEWER)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}

enum Role {
  ADMIN
  MANAGER
  VIEWER
}

model Product {
  id          String   @id @default(cuid())
  name        String
  description String
  price       Float
  inventory   Int
  published   Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

### Permission Matrix

Define what each role can do:

| Action         | Admin | Manager | Viewer |
| -------------- | ----- | ------- | ------ |
| View products  | ✅    | ✅      | ✅     |
| Create product | ✅    | ❌      | ❌     |
| Edit product   | ✅    | ✅      | ❌     |
| Delete product | ✅    | ❌      | ❌     |
| Publish        | ✅    | ✅      | ❌     |
| View orders    | ✅    | ✅      | ✅     |
| Manage users   | ✅    | ❌      | ❌     |

### Permission Helper Functions

Create reusable permission checks:

```typescript
// lib/permissions.ts
import { Session } from 'next-auth';

export type Permission = 
  | 'products:view'
  | 'products:create'
  | 'products:edit'
  | 'products:delete'
  | 'products:publish'
  | 'orders:view'
  | 'users:manage';

const rolePermissions: Record<string, Permission[]> = {
  ADMIN: [
    'products:view',
    'products:create',
    'products:edit',
    'products:delete',
    'products:publish',
    'orders:view',
    'users:manage',
  ],
  MANAGER: [
    'products:view',
    'products:edit',
    'products:publish',
    'orders:view',
  ],
  VIEWER: [
    'products:view',
    'orders:view',
  ],
};

export function hasPermission(
  session: Session | null,
  permission: Permission
): boolean {
  if (!session) return false;
  
  const userRole = session.user.role;
  const permissions = rolePermissions[userRole] || [];
  
  return permissions.includes(permission);
}

export function requirePermission(
  session: Session | null,
  permission: Permission
): void {
  if (!hasPermission(session, permission)) {
    throw new Error(`Missing permission: ${permission}`);
  }
}

export function hasAnyPermission(
  session: Session | null,
  permissions: Permission[]
): boolean {
  return permissions.some(p => hasPermission(session, p));
}

export function hasAllPermissions(
  session: Session | null,
  permissions: Permission[]
): boolean {
  return permissions.every(p => hasPermission(session, p));
}
```

### Protected Product List Page

Show different UI based on permissions:

```tsx
// app/admin/products/page.tsx
import { auth } from '@/lib/auth';
import { hasPermission } from '@/lib/permissions';
import { redirect } from 'next/navigation';
import Link from 'next/link';

async function getProducts() {
  // In production: fetch from database
  return [
    { id: '1', name: 'Product 1', price: 29.99, inventory: 100, published: true },
    { id: '2', name: 'Product 2', price: 49.99, inventory: 50, published: false },
    { id: '3', name: 'Product 3', price: 19.99, inventory: 200, published: true },
  ];
}

export default async function ProductsPage() {
  const session = await auth();

  if (!session) {
    redirect('/login');
  }

  // Check if user can view products
  if (!hasPermission(session, 'products:view')) {
    redirect('/unauthorized');
  }

  const products = await getProducts();
  const canCreate = hasPermission(session, 'products:create');
  const canEdit = hasPermission(session, 'products:edit');
  const canDelete = hasPermission(session, 'products:delete');

  return (
    <div>
      <div className="flex justify-between items-center mb-8">
        <h1 className="text-3xl font-bold">Products</h1>
        {canCreate && (
          <Link
            href="/admin/products/new"
            className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
          >
            Create Product
          </Link>
        )}
      </div>

      <div className="grid gap-4">
        {products.map((product) => (
          <div key={product.id} className="border p-4 rounded flex justify-between items-center">
            <div>
              <h2 className="text-xl font-bold">{product.name}</h2>
              <p className="text-gray-600">Price: ${product.price}</p>
              <p className="text-gray-600">Stock: {product.inventory}</p>
              <span className={`inline-block px-2 py-1 text-xs rounded ${
                product.published 
                  ? 'bg-green-100 text-green-800' 
                  : 'bg-gray-100 text-gray-800'
              }`}>
                {product.published ? 'Published' : 'Draft'}
              </span>
            </div>

            <div className="flex gap-2">
              {canEdit && (
                <Link
                  href={`/admin/products/${product.id}/edit`}
                  className="px-3 py-1 bg-blue-600 text-white rounded hover:bg-blue-700"
                >
                  Edit
                </Link>
              )}
              {canDelete && (
                <button
                  className="px-3 py-1 bg-red-600 text-white rounded hover:bg-red-700"
                >
                  Delete
                </button>
              )}
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**What each role sees**:

**Admin**:
- "Create Product" button visible
- "Edit" button on each product
- "Delete" button on each product

**Manager**:
- No "Create Product" button
- "Edit" button on each product
- No "Delete" button

**Viewer**:
- No "Create Product" button
- No "Edit" button
- No "Delete" button
- Read-only view

### Protected Server Actions

Implement role-based Server Actions:

```typescript
// app/admin/products/actions.ts
'use server';

import { auth } from '@/lib/auth';
import { requirePermission } from '@/lib/permissions';
import { revalidatePath } from 'next/cache';

export async function createProduct(formData: FormData) {
  const session = await auth();
  requirePermission(session, 'products:create');

  const name = formData.get('name') as string;
  const price = parseFloat(formData.get('price') as string);
  const inventory = parseInt(formData.get('inventory') as string);

  // Validate
  if (!name || !price || !inventory) {
    throw new Error('Missing required fields');
  }

  // In production: await db.product.create({ data: { name, price, inventory } })
  console.log('Creating product:', { name, price, inventory });

  revalidatePath('/admin/products');
  return { success: true };
}

export async function updateProduct(id: string, formData: FormData) {
  const session = await auth();
  requirePermission(session, 'products:edit');

  const name = formData.get('name') as string;
  const price = parseFloat(formData.get('price') as string);
  const inventory = parseInt(formData.get('inventory') as string);

  // In production: await db.product.update({ where: { id }, data: { name, price, inventory } })
  console.log('Updating product:', id, { name, price, inventory });

  revalidatePath('/admin/products');
  revalidatePath(`/admin/products/${id}`);
  return { success: true };
}

export async function deleteProduct(id: string) {
  const session = await auth();
  requirePermission(session, 'products:delete');

  // In production: await db.product.delete({ where: { id } })
  console.log('Deleting product:', id);

  revalidatePath('/admin/products');
  return { success: true };
}

export async function publishProduct(id: string) {
  const session = await auth();
  requirePermission(session, 'products:publish');

  // In production: await db.product.update({ where: { id }, data: { published: true } })
  console.log('Publishing product:', id);

  revalidatePath('/admin/products');
  revalidatePath(`/admin/products/${id}`);
  return { success: true };
}
```

### The Failure: Client-Side Permission Checks

What if we only check permissions in the UI?

```tsx
// ❌ BAD - Only hiding the button
export default async function ProductsPage() {
  const session = await auth();
  const canDelete = hasPermission(session, 'products:delete');

  return (
    <div>
      {canDelete && (
        <button onClick={() => deleteProduct(productId)}>
          Delete
        </button>
      )}
    </div>
  );
}
```

**Attack**: Manager opens DevTools and runs:

```javascript
// In browser console
fetch('/api/actions', {
  method: 'POST',
  body: JSON.stringify({
    action: 'deleteProduct',
    args: ['product-123']
  })
});
```

**If Server Action doesn't check permissions**:
```
Product deleted successfully
```

**Diagnostic Analysis**:

**Browser Console Output**:
```
POST /api/actions 200 OK
Response: { success: true }
```

**Network Tab**:
- Request to Server Action succeeds
- No permission check on server

**Root cause**: UI hides the button, but Server Action is still callable.

**Solution**: Always check permissions in Server Actions (as shown above with `requirePermission`).

### Protected API Routes

Protect API endpoints with the same permission system:

```typescript
// app/api/admin/products/route.ts
import { auth } from '@/lib/auth';
import { hasPermission } from '@/lib/permissions';
import { NextResponse } from 'next/server';

export async function GET() {
  const session = await auth();

  if (!hasPermission(session, 'products:view')) {
    return NextResponse.json(
      { error: 'Forbidden' },
      { status: 403 }
    );
  }

  // Fetch products
  const products = [
    { id: '1', name: 'Product 1', price: 29.99 },
  ];

  return NextResponse.json(products);
}

export async function POST(request: Request) {
  const session = await auth();

  if (!hasPermission(session, 'products:create')) {
    return NextResponse.json(
      { error: 'Forbidden' },
      { status: 403 }
    );
  }

  const body = await request.json();

  // Create product
  // await db.product.create({ data: body })

  return NextResponse.json({ success: true });
}
```

```typescript
// app/api/admin/products/[id]/route.ts
import { auth } from '@/lib/auth';
import { hasPermission } from '@/lib/permissions';
import { NextResponse } from 'next/server';

export async function PATCH(
  request: Request,
  { params }: { params: { id: string } }
) {
  const session = await auth();

  if (!hasPermission(session, 'products:edit')) {
    return NextResponse.json(
      { error: 'Forbidden' },
      { status: 403 }
    );
  }

  const body = await request.json();

  // Update product
  // await db.product.update({ where: { id: params.id }, data: body })

  return NextResponse.json({ success: true });
}

export async function DELETE(
  request: Request,
  { params }: { params: { id: string } }
) {
  const session = await auth();

  if (!hasPermission(session, 'products:delete')) {
    return NextResponse.json(
      { error: 'Forbidden' },
      { status: 403 }
    );
  }

  // Delete product
  // await db.product.delete({ where: { id: params.id } })

  return NextResponse.json({ success: true });
}
```

### Testing Role-Based Access

Create test users with different roles:

```typescript
// lib/auth.ts - Update getUserFromDatabase
async function getUserFromDatabase(email: string) {
  const users = {
    'admin@example.com': {
      id: '1',
      email: 'admin@example.com',
      name: 'Admin User',
      role: 'ADMIN',
      passwordHash: '$2a$10$...' // bcrypt hash of 'password123'
    },
    'manager@example.com': {
      id: '2',
      email: 'manager@example.com',
      name: 'Manager User',
      role: 'MANAGER',
      passwordHash: '$2a$10$...'
    },
    'viewer@example.com': {
      id: '3',
      email: 'viewer@example.com',
      name: 'Viewer User',
      role: 'VIEWER',
      passwordHash: '$2a$10$...'
    },
  };

  return users[email] || null;
}
```

**Test scenarios**:

**Test 1: Admin can delete**
1. Log in as admin@example.com
2. Visit `/admin/products`
3. Click "Delete" on a product
4. **Expected**: Product deleted, success message
5. **Actual**: ✅ Works

**Test 2: Manager cannot delete**
1. Log in as manager@example.com
2. Visit `/admin/products`
3. "Delete" button not visible
4. Try to call Server Action directly in console:

```javascript
deleteProduct('product-123');
```

5. **Expected**: Error "Missing permission: products:delete"
6. **Actual**: ✅ Error thrown

**Test 3: Viewer cannot edit**
1. Log in as viewer@example.com
2. Visit `/admin/products`
3. No "Edit" or "Delete" buttons visible
4. Try to visit `/admin/products/1/edit` directly
5. **Expected**: Redirect to `/unauthorized`
6. **Actual**: ✅ Redirected

### Advanced Pattern: Resource-Level Permissions

Sometimes permissions depend on the specific resource:

```typescript
// lib/permissions.ts
export async function canEditProduct(
  session: Session | null,
  productId: string
): Promise<boolean> {
  if (!session) return false;

  // Admins can edit any product
  if (session.user.role === 'ADMIN') {
    return true;
  }

  // Managers can only edit their own products
  if (session.user.role === 'MANAGER') {
    // In production: check if user created this product
    // const product = await db.product.findUnique({
    //   where: { id: productId },
    //   select: { createdById: true }
    // });
    // return product?.createdById === session.user.id;
    
    return true; // Simplified for demo
  }

  return false;
}
```

Use in Server Actions:

```typescript
// app/admin/products/actions.ts
export async function updateProduct(id: string, formData: FormData) {
  const session = await auth();

  if (!await canEditProduct(session, id)) {
    throw new Error('You cannot edit this product');
  }

  // Update product
}
```

### Common Failure Modes and Their Signatures

#### Symptom: User can see UI elements they can't use

**Browser behavior**:
- Edit button visible
- Clicking it shows "Forbidden" error

**Console pattern**:
```
POST /api/admin/products/123 403 Forbidden
{ error: "Forbidden" }
```

**Root cause**: UI permission check missing or incorrect

**Solution**: Check permissions before rendering UI elements:

```tsx
{hasPermission(session, 'products:edit') && (
  <button>Edit</button>
)}
```

#### Symptom: Permission check passes but action fails

**Browser behavior**:
- Button visible and clickable
- Action fails with "Unauthorized"

**Console pattern**:
```
Error: Missing permission: products:delete
```

**Root cause**: UI checks different permission than Server Action

**Solution**: Use the same permission constants everywhere:

```typescript
// ✅ GOOD - Same permission constant
const canDelete = hasPermission(session, 'products:delete');

// In Server Action
requirePermission(session, 'products:delete');
```

#### Symptom: Middleware allows access but page denies it

**Browser behavior**:
- Page loads
- Shows "Unauthorized" message

**Console pattern**:
```
GET /admin/products 200 OK
(Page renders with "Unauthorized" message)
```

**Root cause**: Middleware checks role, page checks specific permission

**Solution**: Align middleware and page checks:

```typescript
// middleware.ts - Check role
if (path.startsWith('/admin') && session.user.role !== 'ADMIN') {
  return NextResponse.redirect(new URL('/unauthorized', req.url));
}

// page.tsx - Check specific permission
if (!hasPermission(session, 'products:view')) {
  redirect('/unauthorized');
}
```

### The Complete Journey: From Broken to Secure

| Iteration | Approach                  | Vulnerability                    | Fix                          |
| --------- | ------------------------- | -------------------------------- | ---------------------------- |
| 0         | Client-only auth          | Everything exposed               | Move to server               |
| 1         | Server Components         | API routes unprotected           | Add API validation           |
| 2         | Middleware                | Server Actions unprotected       | Check in every action        |
| 3         | Role-based UI             | Permissions not enforced         | Add permission system        |
| 4         | Permission system         | UI and server checks misaligned  | Use same permission checks   |
| 5         | Resource-level (current)  | All managers can edit everything | Check resource ownership too |

### Final Implementation: Production-Ready Auth

Here's the complete, secure implementation:

```typescript
// middleware.ts - First line of defense
import { auth } from '@/lib/auth';
import { NextResponse } from 'next/server';

export default auth((req) => {
  const session = req.auth;
  const path = req.nextUrl.pathname;

  // Public routes
  if (path === '/' || path.startsWith('/login')) {
    return NextResponse.next();
  }

  // Require authentication
  if (!session) {
    const loginUrl = new URL('/login', req.url);
    loginUrl.searchParams.set('callbackUrl', path);
    return NextResponse.redirect(loginUrl);
  }

  // Admin routes require admin role
  if (path.startsWith('/admin')) {
    if (!['ADMIN', 'MANAGER', 'VIEWER'].includes(session.user.role)) {
      return NextResponse.redirect(new URL('/unauthorized', req.url));
    }
  }

  return NextResponse.next();
});

export const config = {
  matcher: ['/admin/:path*', '/api/admin/:path*'],
};
```

```tsx
// app/admin/products/page.tsx - Second line of defense
import { auth } from '@/lib/auth';
import { hasPermission } from '@/lib/permissions';
import { redirect } from 'next/navigation';

export default async function ProductsPage() {
  const session = await auth();

  if (!session) {
    redirect('/login');
  }

  if (!hasPermission(session, 'products:view')) {
    redirect('/unauthorized');
  }

  const canCreate = hasPermission(session, 'products:create');
  const canEdit = hasPermission(session, 'products:edit');
  const canDelete = hasPermission(session, 'products:delete');

  // Render UI based on permissions
  return (
    <div>
      {canCreate && <CreateButton />}
      {products.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          canEdit={canEdit}
          canDelete={canDelete}
        />
      ))}
    </div>
  );
}
```

```typescript
// app/admin/products/actions.ts - Third line of defense
'use server';

import { auth } from '@/lib/auth';
import { requirePermission, canEditProduct } from '@/lib/permissions';

export async function deleteProduct(id: string) {
  const session = await auth();
  requirePermission(session, 'products:delete');

  // Additional resource-level check
  if (!await canEditProduct(session, id)) {
    throw new Error('Cannot delete this product');
  }

  // Delete product
  await db.product.delete({ where: { id } });
  revalidatePath('/admin/products');
  return { success: true };
}
```

```typescript
// app/api/admin/products/[id]/route.ts - Fourth line of defense
import { auth } from '@/lib/auth';
import { hasPermission, canEditProduct } from '@/lib/permissions';
import { NextResponse } from 'next/server';

export async function DELETE(
  request: Request,
  { params }: { params: { id: string } }
) {
  const session = await auth();

  if (!hasPermission(session, 'products:delete')) {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
  }

  if (!await canEditProduct(session, params.id)) {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
  }

  await db.product.delete({ where: { id: params.id } });
  return NextResponse.json({ success: true });
}
```

### Decision Framework: When to Use Which Auth Pattern

| Scenario                          | Pattern                    | Why                                      |
| --------------------------------- | -------------------------- | ---------------------------------------- |
| Protect entire route tree         | Middleware                 | Runs before any code executes            |
| Show/hide UI elements             | Client-side permission     | Better UX, but not security              |
| Protect data mutations            | Server Action validation   | Mutations must always validate           |
| Protect API endpoints             | API route validation       | External access requires validation      |
| Resource-specific permissions     | Resource-level checks      | Permissions depend on specific data      |
| Multi-tenant applications         | Tenant-scoped queries      | Isolate data by tenant ID                |
| Temporary access (share links)    | Time-limited tokens        | Token expires after set duration         |
| Third-party API access            | API keys + rate limiting   | Different auth mechanism for external    |
| Audit trail requirements          | Log all permission checks  | Track who accessed what and when         |

### Lessons Learned

**1. Defense in depth**: Check permissions at every layer
- Middleware: Protect routes
- Page: Check before rendering
- Server Action: Validate before mutation
- API Route: Validate before data access

**2. Client-side checks are UX, not security**: Always validate on the server

**3. Use a permission system**: Don't hardcode role checks everywhere

**4. Test with different roles**: Create test users for each role

**5. Resource-level permissions matter**: Not all admins should access all resources

**6. Audit and log**: Track permission checks for security analysis

**7. Keep middleware fast**: No database queries or external API calls

**8. Align UI and server checks**: Use the same permission constants

The journey from broken client-side auth to production-ready role-based access control is about understanding that security is not a single check—it's a layered system where each layer validates independently, and the UI is merely a reflection of what the server allows.
