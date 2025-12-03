# Chapter 14: Advanced Routing Patterns

## Code splitting and lazy loading

## The Problem: Everything Loads at Once

In Chapter 13, we built a multi-page documentation site with React Router. It works—users can navigate between pages without full page reloads. But there's a hidden performance problem that becomes obvious as the application grows.

Let's establish our reference implementation: a documentation site with multiple sections. This will be our anchor example throughout this chapter.

**Project Structure**:
```
src/
├── pages/
│   ├── Home.tsx           ← Landing page
│   ├── GettingStarted.tsx ← Tutorial content
│   ├── APIReference.tsx   ← Large API docs
│   ├── Examples.tsx       ← Code examples
│   └── Changelog.tsx      ← Version history
├── components/
│   ├── Navigation.tsx
│   └── Layout.tsx
└── App.tsx               ← Router setup
```

Here's our initial implementation:

```tsx
// src/App.tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import Layout from './components/Layout';
import Home from './pages/Home';
import GettingStarted from './pages/GettingStarted';
import APIReference from './pages/APIReference';
import Examples from './pages/Examples';
import Changelog from './pages/Changelog';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route index element={<Home />} />
          <Route path="getting-started" element={<GettingStarted />} />
          <Route path="api" element={<APIReference />} />
          <Route path="examples" element={<Examples />} />
          <Route path="changelog" element={<Changelog />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}

export default App;
```

## Route-based data loading

## The Problem: Data Fetching Waterfalls

Our routes load quickly now, but there's another performance problem. Let's add data fetching to our API Reference page:

```tsx
// src/pages/APIReference.tsx - Initial Implementation
import { useState, useEffect } from 'react';

interface APIMethod {
  id: string;
  name: string;
  category: string;
  description: string;
  parameters: Array<{
    name: string;
    type: string;
    description: string;
  }>;
  examples: string[];
}

export default function APIReference() {
  const [methods, setMethods] = useState<APIMethod[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetch('/api/methods')
      .then(res => res.json())
      .then(data => {
        setMethods(data);
        setIsLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setIsLoading(false);
      });
  }, []);

  if (isLoading) return <div>Loading API methods...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div className="api-reference">
      <h1>API Reference</h1>
      {methods.map(method => (
        <div key={method.id} className="method">
          <h2>{method.name}</h2>
          <p>{method.description}</p>
          {/* Render method details */}
        </div>
      ))}
    </div>
  );
}
```

Let's see what happens when a user navigates to the API Reference page:

**Browser DevTools - Network Tab**:
```
Timeline:
0.0s: User clicks "API Reference" link
0.0s: Request for APIReference-c2d3e4f5.js starts
1.9s: APIReference-c2d3e4f5.js finishes downloading
1.9s: React renders component, useEffect runs
1.9s: Request for /api/methods starts
3.2s: /api/methods finishes (1.3s response time)
3.2s: Content appears
```

**Browser Console**:
```
[React Router] Navigating to /api
[Suspense] APIReference suspended
[Suspense] APIReference resolved
[Component] APIReference mounted
[Network] GET /api/methods
[Component] APIReference updated with data
```

### Diagnostic Analysis: The Waterfall Problem

**Browser Behavior**:
User clicks "API Reference". They see "Loading page..." for 1.9 seconds. Then they see "Loading API methods..." for another 1.3 seconds. Total wait: 3.2 seconds before seeing content.

**Network Tab Evidence**:
The smoking gun: Sequential requests.
1. First: Download the component code (1.9s)
2. Then: Download the data (1.3s)
3. Total: 3.2 seconds

This is a **waterfall**: The second request can't start until the first completes.

**Performance Metrics**:
- Time to First Byte (TTFB): 1.9s (waiting for component)
- Time to Content: 3.2s (waiting for component + data)
- Largest Contentful Paint (LCP): 3.4s (poor)

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Page loads in ~2 seconds
   - Actual: Two separate loading states, 3.2 seconds total

2. **What the Network tab reveals**:
   - Key indicator: Sequential requests (waterfall pattern)
   - Component code must download before data request starts
   - Each request blocks the next

3. **Root cause identified**: The component code contains the data fetching logic. React can't fetch data until the component code downloads and executes. The data request is blocked by the code request.

4. **Why the current approach can't solve this**: `useEffect` runs after component mounts. Component can't mount until its code downloads. This creates an unavoidable waterfall.

5. **What we need**: A way to start fetching data before the component code downloads. Ideally, both requests should start simultaneously.

### Solution 1: Loader Functions (React Router 6.4+)

React Router 6.4 introduced **loaders**: functions that run before a route renders. They can fetch data in parallel with component code.

```tsx
// src/loaders/apiReferenceLoader.ts
import { LoaderFunctionArgs } from 'react-router-dom';

export interface APIMethod {
  id: string;
  name: string;
  category: string;
  description: string;
  parameters: Array<{
    name: string;
    type: string;
    description: string;
  }>;
  examples: string[];
}

export async function apiReferenceLoader({ params }: LoaderFunctionArgs) {
  const response = await fetch('/api/methods');
  
  if (!response.ok) {
    throw new Error(`Failed to load API methods: ${response.statusText}`);
  }
  
  const methods: APIMethod[] = await response.json();
  return { methods };
}
```

```tsx
// src/pages/APIReference.tsx - Iteration 1: Using Loader
import { useLoaderData } from 'react-router-dom';
import { APIMethod } from '../loaders/apiReferenceLoader';

export default function APIReference() {
  // Data is already loaded by the time component renders
  const { methods } = useLoaderData() as { methods: APIMethod[] };

  return (
    <div className="api-reference">
      <h1>API Reference</h1>
      {methods.map(method => (
        <div key={method.id} className="method">
          <h2>{method.name}</h2>
          <p>{method.description}</p>
          {/* Render method details */}
        </div>
      ))}
    </div>
  );
}
```

```tsx
// src/App.tsx - Iteration 4: Adding Loader to Route
import { 
  createBrowserRouter, 
  RouterProvider,
  Outlet 
} from 'react-router-dom';
import { lazy, Suspense } from 'react';
import Layout from './components/Layout';
import Home from './pages/Home';
import GettingStarted from './pages/GettingStarted';
import { apiReferenceLoader } from './loaders/apiReferenceLoader';

const APIReference = lazy(() => import('./pages/APIReference'));
const Examples = lazy(() => import('./pages/Examples'));
const Changelog = lazy(() => import('./pages/Changelog'));

function RouteLoadingFallback() {
  return (
    <div className="route-loading">
      <div className="spinner" />
      <p>Loading...</p>
    </div>
  );
}

// Create router with loader
const router = createBrowserRouter([
  {
    path: '/',
    element: <Layout />,
    children: [
      {
        index: true,
        element: <Home />
      },
      {
        path: 'getting-started',
        element: <GettingStarted />
      },
      {
        path: 'api',
        element: (
          <Suspense fallback={<RouteLoadingFallback />}>
            <APIReference />
          </Suspense>
        ),
        loader: apiReferenceLoader // ← Loader runs before component renders
      },
      {
        path: 'examples',
        element: (
          <Suspense fallback={<RouteLoadingFallback />}>
            <Examples />
          </Suspense>
        )
      },
      {
        path: 'changelog',
        element: (
          <Suspense fallback={<RouteLoadingFallback />}>
            <Changelog />
          </Suspense>
        )
      }
    ]
  }
]);

function App() {
  return <RouterProvider router={router} />;
}

export default App;
```

Now let's see what happens when a user navigates to the API Reference page:

**Browser DevTools - Network Tab**:
```
Timeline:
0.0s: User clicks "API Reference" link
0.0s: Request for APIReference-c2d3e4f5.js starts
0.0s: Request for /api/methods starts (parallel!)
1.3s: /api/methods finishes
1.9s: APIReference-c2d3e4f5.js finishes
1.9s: Content appears (data already available)
```

**Expected vs. Actual Improvement**:

| Metric | Before (useEffect) | After (Loader) | Improvement |
|--------|-------------------|----------------|-------------|
| Time to Content | 3.2s | 1.9s | 41% faster |
| Number of loading states | 2 | 1 | Simpler UX |
| Waterfall eliminated | No | Yes | Parallel loading |

**What changed**:
1. Loader function runs when navigation starts (before component code downloads)
2. Data request and component code request happen in parallel
3. Component renders only after both complete
4. User sees single loading state, then content

### Handling Loader Errors

Loaders can throw errors. React Router catches them and renders an error boundary:

```tsx
// src/loaders/apiReferenceLoader.ts - Iteration 2: Error Handling
import { LoaderFunctionArgs } from 'react-router-dom';

export async function apiReferenceLoader({ params }: LoaderFunctionArgs) {
  const response = await fetch('/api/methods');
  
  if (!response.ok) {
    // Throw with structured error data
    throw new Response('Failed to load API methods', {
      status: response.status,
      statusText: response.statusText
    });
  }
  
  const methods = await response.json();
  return { methods };
}
```

```tsx
// src/components/ErrorBoundary.tsx
import { useRouteError, isRouteErrorResponse } from 'react-router-dom';

export default function ErrorBoundary() {
  const error = useRouteError();

  if (isRouteErrorResponse(error)) {
    return (
      <div className="error-boundary">
        <h1>{error.status} {error.statusText}</h1>
        <p>{error.data}</p>
      </div>
    );
  }

  return (
    <div className="error-boundary">
      <h1>Oops! Something went wrong</h1>
      <p>{error instanceof Error ? error.message : 'Unknown error'}</p>
    </div>
  );
}
```

```tsx
// src/App.tsx - Adding Error Boundary
const router = createBrowserRouter([
  {
    path: '/',
    element: <Layout />,
    errorElement: <ErrorBoundary />, // ← Catches loader errors
    children: [
      // ... routes
      {
        path: 'api',
        element: (
          <Suspense fallback={<RouteLoadingFallback />}>
            <APIReference />
          </Suspense>
        ),
        loader: apiReferenceLoader
      }
    ]
  }
]);
```

### Loading States with Loaders

Loaders don't show loading states by default. The navigation waits until the loader completes. For slow loaders, this creates a "frozen" UI. Let's add a loading indicator:

```tsx
// src/components/Layout.tsx - Iteration 5: Navigation Loading State
import { Outlet, useNavigation } from 'react-router-dom';
import Navigation from './Navigation';

export default function Layout() {
  const navigation = useNavigation();
  const isLoading = navigation.state === 'loading';

  return (
    <div className="layout">
      <Navigation />
      
      {/* Show loading bar during navigation */}
      {isLoading && (
        <div className="loading-bar">
          <div className="loading-bar-progress" />
        </div>
      )}
      
      <main>
        <Outlet />
      </main>
    </div>
  );
}
```

**Browser Behavior**:
User clicks "API Reference". A loading bar appears at the top of the page. The current page remains visible. After 1.9 seconds, the loading bar disappears and the new page appears.

**Improvement**: User sees immediate feedback (loading bar) while keeping the current page visible. Better than a blank screen or full-page spinner.

### Solution 2: Prefetching Data (Advanced)

For even better performance, we can prefetch data before the user clicks:

```tsx
// src/components/Navigation.tsx - Iteration 6: Prefetch on Hover
import { Link, useNavigate } from 'react-router-dom';
import { apiReferenceLoader } from '../loaders/apiReferenceLoader';

export default function Navigation() {
  const navigate = useNavigate();

  // Prefetch data when user hovers over link
  const handleMouseEnter = () => {
    // Start loading data before user clicks
    apiReferenceLoader({ params: {}, request: new Request('/api') } as any);
  };

  return (
    <nav>
      <Link to="/">Home</Link>
      <Link to="/getting-started">Getting Started</Link>
      <Link 
        to="/api" 
        onMouseEnter={handleMouseEnter} // ← Prefetch on hover
      >
        API Reference
      </Link>
      <Link to="/examples">Examples</Link>
      <Link to="/changelog">Changelog</Link>
    </nav>
  );
}
```

**Browser DevTools - Network Tab** (with prefetch):
```
Timeline:
0.0s: User hovers over "API Reference" link
0.0s: Request for /api/methods starts (prefetch)
0.5s: User clicks link
0.5s: Request for APIReference-c2d3e4f5.js starts
1.3s: /api/methods finishes (already in cache)
1.4s: APIReference-c2d3e4f5.js finishes
1.4s: Content appears instantly (data already loaded)
```

**Expected vs. Actual Improvement**:

| Metric | Loader Only | Loader + Prefetch | Improvement |
|--------|-------------|-------------------|-------------|
| Time to Content | 1.9s | 1.4s | 26% faster |
| Perceived speed | Fast | Instant | Feels immediate |

**When to Apply Prefetching**:

**What it optimizes for**:
- Perceived performance (feels instant)
- User experience on fast connections
- Frequently accessed routes

**What it sacrifices**:
- Bandwidth (may fetch data user doesn't need)
- Server load (more requests)
- Complexity (more code to maintain)

**When to prefetch**:
- High-probability navigation (user hovering over link)
- Fast data endpoints (< 500ms response time)
- Small data payloads (< 100 KB)
- Frequently accessed routes (> 50% of users visit)

**When NOT to prefetch**:
- Slow data endpoints (> 1s response time)
- Large data payloads (> 500 KB)
- Rarely accessed routes
- Mobile users on metered connections
- Data that changes frequently (risk of stale data)

### Combining Loaders with React Query

For more sophisticated data management, combine loaders with React Query:

```tsx
// src/loaders/apiReferenceLoader.ts - Iteration 3: React Query Integration
import { QueryClient } from '@tanstack/react-query';
import { LoaderFunctionArgs } from 'react-router-dom';

const queryClient = new QueryClient();

async function fetchAPIMethods() {
  const response = await fetch('/api/methods');
  if (!response.ok) {
    throw new Error('Failed to load API methods');
  }
  return response.json();
}

export async function apiReferenceLoader({ params }: LoaderFunctionArgs) {
  // Ensure data is in React Query cache
  const methods = await queryClient.ensureQueryData({
    queryKey: ['api-methods'],
    queryFn: fetchAPIMethods,
    staleTime: 5 * 60 * 1000 // 5 minutes
  });
  
  return { methods };
}
```

```tsx
// src/pages/APIReference.tsx - Iteration 2: React Query in Component
import { useQuery } from '@tanstack/react-query';

async function fetchAPIMethods() {
  const response = await fetch('/api/methods');
  if (!response.ok) {
    throw new Error('Failed to load API methods');
  }
  return response.json();
}

export default function APIReference() {
  // Data is already in cache from loader
  const { data: methods } = useQuery({
    queryKey: ['api-methods'],
    queryFn: fetchAPIMethods,
    staleTime: 5 * 60 * 1000
  });

  return (
    <div className="api-reference">
      <h1>API Reference</h1>
      {methods?.map(method => (
        <div key={method.id} className="method">
          <h2>{method.name}</h2>
          <p>{method.description}</p>
        </div>
      ))}
    </div>
  );
}
```

**Benefits of React Query + Loaders**:
1. Loader ensures data is available before render
2. React Query provides caching, revalidation, and refetching
3. Subsequent navigations to the same route are instant (cached)
4. Data stays fresh with automatic background refetching

**This solves the data loading problem, but we still need to handle scroll position...**

## Scroll restoration and focus management

## The Problem: Lost Scroll Position and Focus

Our routes load fast, data loads in parallel, but there's a subtle UX problem. Let's see what happens when users navigate:

**User Journey**:
1. User visits home page
2. Scrolls down to read content (scroll position: 800px)
3. Clicks "API Reference" link
4. Reads API docs, scrolls down (scroll position: 1200px)
5. Clicks browser back button
6. **Expected**: Returns to home page at scroll position 800px
7. **Actual**: Returns to home page at scroll position 0px (top)

**Browser Console**:
```
[React Router] Navigating to /api
[Window] Scroll position: 800px
[React Router] Navigation complete
[Window] Scroll position: 0px (reset!)
```

### Diagnostic Analysis: The Scroll Position Failure

**Browser Behavior**:
User navigates back to a page they were reading. They expect to return to where they left off. Instead, they're scrolled to the top. They must scroll down again to find their place. Frustrating.

**React DevTools Evidence**:
- Component unmounts on navigation
- New component mounts
- Scroll position resets to 0

**Root cause identified**: React Router doesn't restore scroll position by default. When a route changes, the browser scrolls to the top (default behavior). The previous scroll position is lost.

**Why the current approach can't solve this**: React Router has no built-in scroll restoration. The browser's native scroll restoration is disabled in single-page applications. We must implement it ourselves.

**What we need**: A way to:
1. Save scroll position before navigation
2. Restore scroll position after navigation
3. Handle edge cases (scroll to top for new pages, restore for back/forward)

### Solution: Scroll Restoration

React Router provides `ScrollRestoration` component for this:

```tsx
// src/App.tsx - Iteration 7: Adding Scroll Restoration
import { 
  createBrowserRouter, 
  RouterProvider,
  ScrollRestoration
} from 'react-router-dom';
import Layout from './components/Layout';

const router = createBrowserRouter([
  {
    path: '/',
    element: (
      <>
        <ScrollRestoration /> {/* ← Handles scroll restoration */}
        <Layout />
      </>
    ),
    children: [
      // ... routes
    ]
  }
]);

function App() {
  return <RouterProvider router={router} />;
}

export default App;
```

**Browser Behavior** (after adding ScrollRestoration):
1. User scrolls to 800px on home page
2. Navigates to API Reference
3. Scrolls to 1200px
4. Clicks back button
5. **Result**: Returns to home page at scroll position 800px ✓

**How it works**:
- `ScrollRestoration` saves scroll position in `sessionStorage` before navigation
- After navigation, it restores the saved position
- For forward navigation (clicking links), it scrolls to top
- For back/forward navigation (browser buttons), it restores position

### Customizing Scroll Behavior

Sometimes you want different behavior:

```tsx
// src/App.tsx - Iteration 8: Custom Scroll Behavior
import { 
  createBrowserRouter, 
  RouterProvider,
  ScrollRestoration
} from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: (
      <>
        <ScrollRestoration 
          getKey={(location, matches) => {
            // Custom key for scroll position storage
            // Same key = restore position, different key = scroll to top
            
            // For API Reference, restore position even on forward navigation
            if (location.pathname === '/api') {
              return location.pathname;
            }
            
            // For other routes, use default behavior
            return location.key;
          }}
        />
        <Layout />
      </>
    ),
    children: [
      // ... routes
    ]
  }
]);
```

### Scrolling to Top on Route Change

For some routes, you always want to scroll to top:

```tsx
// src/hooks/useScrollToTop.ts
import { useEffect } from 'react';
import { useLocation } from 'react-router-dom';

export function useScrollToTop() {
  const { pathname } = useLocation();

  useEffect(() => {
    window.scrollTo(0, 0);
  }, [pathname]);
}
```

```tsx
// src/pages/Examples.tsx - Using useScrollToTop
import { useScrollToTop } from '../hooks/useScrollToTop';

export default function Examples() {
  useScrollToTop(); // ← Always scroll to top when this route renders

  return (
    <div className="examples">
      <h1>Code Examples</h1>
      {/* Content */}
    </div>
  );
}
```

### The Focus Management Problem

Scroll position is only half the story. Let's see what happens with keyboard navigation:

**User Journey** (keyboard user):
1. User tabs through navigation links
2. Presses Enter on "API Reference" (focus on link)
3. Page changes to API Reference
4. **Expected**: Focus moves to main content (for screen reader announcement)
5. **Actual**: Focus stays on navigation link (now invisible/wrong page)

**Browser Console**:
```
[Focus] Active element: <a href="/api">API Reference</a>
[React Router] Navigating to /api
[Focus] Active element: <a href="/api">API Reference</a> (still!)
```

### Diagnostic Analysis: The Focus Management Failure

**Browser Behavior**:
Keyboard user navigates to a new page. Focus remains on the navigation link. Screen reader doesn't announce the new page. User must tab through navigation again to reach content. Poor accessibility.

**Accessibility Audit** (Lighthouse):
```
[Accessibility] Focus not managed on route change
[Accessibility] Screen reader users may not know page changed
Score: 78/100 (down from 95)
```

**Root cause identified**: React Router doesn't manage focus on route changes. The browser keeps focus on the element that triggered navigation (the link). This breaks keyboard navigation and screen reader experience.

**Why the current approach can't solve this**: Focus management requires explicit code. We must programmatically move focus to the new content.

**What we need**: A way to move focus to the main content area when the route changes, so screen readers announce the new page and keyboard users can immediately interact with content.

### Solution: Focus Management

```tsx
// src/components/Layout.tsx - Iteration 9: Focus Management
import { Outlet, useLocation } from 'react-router-dom';
import { useEffect, useRef } from 'react';
import Navigation from './Navigation';

export default function Layout() {
  const location = useLocation();
  const mainRef = useRef<HTMLElement>(null);

  // Move focus to main content on route change
  useEffect(() => {
    if (mainRef.current) {
      mainRef.current.focus();
    }
  }, [location.pathname]);

  return (
    <div className="layout">
      <Navigation />
      
      <main 
        ref={mainRef}
        tabIndex={-1} // ← Makes element focusable
        className="main-content"
      >
        <Outlet />
      </main>
    </div>
  );
}
```

**Browser Behavior** (after adding focus management):
1. User presses Enter on "API Reference" link
2. Page changes
3. Focus moves to `<main>` element
4. Screen reader announces: "API Reference, main content"
5. User can immediately interact with content

**Accessibility Audit** (after fix):
```
[Accessibility] Focus managed correctly on route change
[Accessibility] Screen reader announces page changes
Score: 95/100 ✓
```

### Skip Links for Keyboard Users

For even better accessibility, add a skip link:

```tsx
// src/components/Layout.tsx - Iteration 10: Skip Link
import { Outlet, useLocation } from 'react-router-dom';
import { useEffect, useRef } from 'react';
import Navigation from './Navigation';

export default function Layout() {
  const location = useLocation();
  const mainRef = useRef<HTMLElement>(null);

  useEffect(() => {
    if (mainRef.current) {
      mainRef.current.focus();
    }
  }, [location.pathname]);

  return (
    <div className="layout">
      {/* Skip link - visible only on keyboard focus */}
      <a 
        href="#main-content" 
        className="skip-link"
        onClick={(e) => {
          e.preventDefault();
          mainRef.current?.focus();
        }}
      >
        Skip to main content
      </a>
      
      <Navigation />
      
      <main 
        id="main-content"
        ref={mainRef}
        tabIndex={-1}
        className="main-content"
      >
        <Outlet />
      </main>
    </div>
  );
}
```

```css
/* src/styles/layout.css */
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: #fff;
  padding: 8px;
  text-decoration: none;
  z-index: 100;
}

.skip-link:focus {
  top: 0; /* Becomes visible when focused */
}
```

**User Experience**:
Keyboard user presses Tab on page load. Skip link appears at top of page. User presses Enter. Focus jumps directly to main content, bypassing navigation. Saves time and keystrokes.

### Announcing Route Changes to Screen Readers

For screen reader users, we can add live region announcements:

```tsx
// src/components/RouteAnnouncer.tsx
import { useEffect, useRef } from 'react';
import { useLocation } from 'react-router-dom';

export default function RouteAnnouncer() {
  const location = useLocation();
  const announceRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    // Extract page title from pathname
    const pageName = location.pathname
      .split('/')
      .filter(Boolean)
      .join(' ')
      .replace(/-/g, ' ') || 'home';
    
    // Announce to screen readers
    if (announceRef.current) {
      announceRef.current.textContent = `Navigated to ${pageName} page`;
    }
  }, [location.pathname]);

  return (
    <div
      ref={announceRef}
      role="status"
      aria-live="polite"
      aria-atomic="true"
      className="sr-only" // Visually hidden, but announced
    />
  );
}
```

```tsx
// src/App.tsx - Adding Route Announcer
import RouteAnnouncer from './components/RouteAnnouncer';

const router = createBrowserRouter([
  {
    path: '/',
    element: (
      <>
        <ScrollRestoration />
        <RouteAnnouncer /> {/* ← Announces route changes */}
        <Layout />
      </>
    ),
    children: [
      // ... routes
    ]
  }
]);
```

```css
/* src/styles/accessibility.css */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}
```

**Screen Reader Experience**:
User navigates to API Reference page. Screen reader announces: "Navigated to API Reference page." User knows the navigation succeeded without visual confirmation.

### When to Apply Scroll and Focus Management

**What it optimizes for**:
- User experience (returning to where they left off)
- Accessibility (keyboard and screen reader users)
- Professional polish (attention to detail)

**What it sacrifices**:
- Slightly more complex code
- Need to test with keyboard and screen readers

**When to apply**:
- All production applications (accessibility is not optional)
- Long-form content (articles, documentation)
- Applications with deep navigation hierarchies
- Any application used by keyboard or screen reader users

**When NOT to apply**:
- Never. Always implement scroll restoration and focus management.

**Code characteristics**:
- Setup complexity: Low (built-in components + a few hooks)
- Maintenance burden: Very low (set it and forget it)
- Accessibility impact: Critical (makes app usable for all users)

**This solves scroll and focus, but we still need to optimize navigation performance...**

## When to prefetch

## The Problem: Slow Navigation on Slow Connections

Our application loads efficiently, but on slow connections, navigation still feels sluggish. Let's see what happens when a user on 3G tries to navigate:

**Browser DevTools - Network Tab** (throttled to 3G):
```
Timeline:
0.0s: User clicks "Examples" link
0.0s: Request for Examples-d3e4f5g6.js starts
2.1s: Examples-d3e4f5g6.js finishes downloading
2.1s: Content appears
```

**User Experience**:
User clicks link. Loading indicator appears. They wait 2.1 seconds. Content appears. On fast connections, this is fine. On slow connections, it feels broken.

### Diagnostic Analysis: The Slow Navigation Problem

**Browser Behavior**:
User on mobile 3G connection clicks a link. They see a loading indicator for 2+ seconds. The wait feels long. They wonder if the click registered. They might click again (double navigation).

**Network Tab Evidence**:
- 3G connection: 750 Kbps download speed
- Examples chunk: 312 KB (105 KB gzipped)
- Download time: 2.1 seconds
- Parse/compile time: 0.3 seconds
- Total: 2.4 seconds to interactive

**Performance Metrics**:
- Time to Interactive: 2.4s (poor on 3G)
- User frustration: High
- Perceived performance: Slow

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Instant or near-instant navigation
   - Actual: 2+ second wait on slow connections

2. **What the Network tab reveals**:
   - Key indicator: Large chunk size + slow connection = long wait
   - Download time dominates (2.1s of 2.4s total)

3. **Root cause identified**: Code splitting creates on-demand loading. On slow connections, "on-demand" means "wait 2+ seconds." The user must wait for the chunk to download before seeing content.

4. **Why the current approach can't solve this**: Lazy loading is reactive. It starts loading when the user clicks. On slow connections, this creates noticeable delay.

5. **What we need**: Proactive loading. Start downloading chunks before the user clicks, so they're ready when needed.

### Solution: Link Prefetching

Prefetching loads resources before they're needed. There are several strategies:

### Strategy 1: Prefetch on Hover

Load the chunk when the user hovers over a link:

```tsx
// src/components/PrefetchLink.tsx
import { Link, LinkProps } from 'react-router-dom';
import { useEffect, useRef, useState } from 'react';

interface PrefetchLinkProps extends LinkProps {
  prefetch?: 'hover' | 'intent' | 'render' | 'none';
}

export default function PrefetchLink({ 
  prefetch = 'hover',
  to,
  children,
  ...props 
}: PrefetchLinkProps) {
  const [isPrefetched, setIsPrefetched] = useState(false);
  const timeoutRef = useRef<number>();

  const handleMouseEnter = () => {
    if (isPrefetched || prefetch === 'none') return;

    // Wait 100ms before prefetching (user might just be passing through)
    timeoutRef.current = window.setTimeout(() => {
      prefetchRoute(to.toString());
      setIsPrefetched(true);
    }, 100);
  };

  const handleMouseLeave = () => {
    // Cancel prefetch if user moves away quickly
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }
  };

  useEffect(() => {
    // Prefetch on render if specified
    if (prefetch === 'render') {
      prefetchRoute(to.toString());
      setIsPrefetched(true);
    }

    return () => {
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
    };
  }, [prefetch, to]);

  return (
    <Link
      to={to}
      onMouseEnter={handleMouseEnter}
      onMouseLeave={handleMouseLeave}
      {...props}
    >
      {children}
    </Link>
  );
}

// Prefetch route chunk
function prefetchRoute(path: string) {
  // This is a simplified version - in production, you'd use React Router's
  // internal prefetching or a more sophisticated approach
  const routeMap: Record<string, () => Promise<any>> = {
    '/examples': () => import('../pages/Examples'),
    '/api': () => import('../pages/APIReference'),
    '/changelog': () => import('../pages/Changelog')
  };

  const prefetchFn = routeMap[path];
  if (prefetchFn) {
    prefetchFn().catch(() => {
      // Silently fail - user will load on click if prefetch fails
    });
  }
}
```

```tsx
// src/components/Navigation.tsx - Using PrefetchLink
import PrefetchLink from './PrefetchLink';

export default function Navigation() {
  return (
    <nav>
      <PrefetchLink to="/">Home</PrefetchLink>
      <PrefetchLink to="/getting-started">Getting Started</PrefetchLink>
      <PrefetchLink to="/api" prefetch="hover">
        API Reference
      </PrefetchLink>
      <PrefetchLink to="/examples" prefetch="hover">
        Examples
      </PrefetchLink>
      <PrefetchLink to="/changelog">Changelog</PrefetchLink>
    </nav>
  );
}
```

**Browser DevTools - Network Tab** (with hover prefetch):
```
Timeline:
0.0s: User hovers over "Examples" link
0.1s: Request for Examples-d3e4f5g6.js starts (prefetch)
0.5s: User clicks link
2.2s: Examples-d3e4f5g6.js finishes downloading
2.2s: Content appears (chunk already loaded!)
```

**Expected vs. Actual Improvement**:

| Metric | No Prefetch | Hover Prefetch | Improvement |
|--------|-------------|----------------|-------------|
| Time to Interactive (3G) | 2.4s | 0.2s | 92% faster |
| Perceived speed | Slow | Instant | Feels immediate |
| Wasted bandwidth | 0 KB | ~10 KB (hover without click) | Minimal |

**How it works**:
1. User hovers over link
2. After 100ms delay (to avoid prefetching on accidental hovers), prefetch starts
3. Chunk downloads in background
4. When user clicks, chunk is already loaded
5. Navigation feels instant

### Strategy 2: Prefetch on Viewport Visibility

Load chunks when links become visible:

```tsx
// src/hooks/useIntersectionPrefetch.ts
import { useEffect, useRef } from 'react';

export function useIntersectionPrefetch(
  prefetchFn: () => void,
  enabled: boolean = true
) {
  const ref = useRef<HTMLElement>(null);
  const hasPrefetched = useRef(false);

  useEffect(() => {
    if (!enabled || hasPrefetched.current) return;

    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting && !hasPrefetched.current) {
            prefetchFn();
            hasPrefetched.current = true;
          }
        });
      },
      {
        rootMargin: '50px' // Start prefetching 50px before element is visible
      }
    );

    if (ref.current) {
      observer.observe(ref.current);
    }

    return () => {
      observer.disconnect();
    };
  }, [prefetchFn, enabled]);

  return ref;
}
```

```tsx
// src/components/Navigation.tsx - Using Intersection Prefetch
import { useIntersectionPrefetch } from '../hooks/useIntersectionPrefetch';

export default function Navigation() {
  const examplesLinkRef = useIntersectionPrefetch(
    () => import('../pages/Examples'),
    true
  );

  return (
    <nav>
      <Link to="/">Home</Link>
      <Link to="/getting-started">Getting Started</Link>
      <Link to="/api">API Reference</Link>
      <Link 
        to="/examples" 
        ref={examplesLinkRef as any}
      >
        Examples
      </Link>
      <Link to="/changelog">Changelog</Link>
    </nav>
  );
}
```

**When to use viewport prefetching**:
- Links in footer or below the fold
- Long pages with navigation at bottom
- Mobile layouts where navigation is collapsed

### Strategy 3: Prefetch on Idle

Load chunks when the browser is idle:

```tsx
// src/hooks/useIdlePrefetch.ts
import { useEffect } from 'react';

export function useIdlePrefetch(
  prefetchFn: () => void,
  enabled: boolean = true
) {
  useEffect(() => {
    if (!enabled) return;

    // Use requestIdleCallback if available, otherwise setTimeout
    const idleCallback = 'requestIdleCallback' in window
      ? window.requestIdleCallback
      : (cb: () => void) => setTimeout(cb, 1);

    const handle = idleCallback(() => {
      prefetchFn();
    });

    return () => {
      if ('cancelIdleCallback' in window) {
        window.cancelIdleCallback(handle as number);
      } else {
        clearTimeout(handle as number);
      }
    };
  }, [prefetchFn, enabled]);
}
```

```tsx
// src/App.tsx - Prefetch Heavy Routes on Idle
import { useIdlePrefetch } from './hooks/useIdlePrefetch';

function App() {
  // Prefetch heavy routes when browser is idle
  useIdlePrefetch(() => {
    import('./pages/APIReference');
    import('./pages/Examples');
  });

  return <RouterProvider router={router} />;
}
```

**When to use idle prefetching**:
- Heavy routes that most users will visit
- After initial page load completes
- When you want to optimize for subsequent navigation
- Desktop users with fast connections

### Strategy 4: Prefetch on Intent (Advanced)

Predict user intent and prefetch accordingly:

```tsx
// src/hooks/useIntentPrefetch.ts
import { useEffect, useRef } from 'react';

interface IntentPrefetchOptions {
  prefetchFn: () => void;
  enabled?: boolean;
  threshold?: number; // Mouse movement threshold in pixels
}

export function useIntentPrefetch({
  prefetchFn,
  enabled = true,
  threshold = 20
}: IntentPrefetchOptions) {
  const ref = useRef<HTMLElement>(null);
  const hasPrefetched = useRef(false);
  const mousePosition = useRef({ x: 0, y: 0 });

  useEffect(() => {
    if (!enabled || hasPrefetched.current) return;

    const element = ref.current;
    if (!element) return;

    const handleMouseMove = (e: MouseEvent) => {
      const rect = element.getBoundingClientRect();
      const dx = e.clientX - mousePosition.current.x;
      const dy = e.clientY - mousePosition.current.y;
      
      mousePosition.current = { x: e.clientX, y: e.clientY };

      // Check if mouse is moving toward the element
      const isMovingToward = 
        e.clientX >= rect.left - threshold &&
        e.clientX <= rect.right + threshold &&
        e.clientY >= rect.top - threshold &&
        e.clientY <= rect.bottom + threshold &&
        (dx > 0 || dy > 0); // Moving right or down

      if (isMovingToward && !hasPrefetched.current) {
        prefetchFn();
        hasPrefetched.current = true;
      }
    };

    document.addEventListener('mousemove', handleMouseMove);

    return () => {
      document.removeEventListener('mousemove', handleMouseMove);
    };
  }, [prefetchFn, enabled, threshold]);

  return ref;
}
```

**When to use intent prefetching**:
- High-value routes (checkout, dashboard)
- When hover prefetching is too aggressive
- Desktop applications with mouse interaction
- When you want to minimize wasted bandwidth

### The Failure: Prefetching Everything

Let's see what happens when we prefetch too aggressively:

```tsx
// src/App.tsx - ANTI-PATTERN: Prefetch Everything
import { useEffect } from 'react';

function App() {
  useEffect(() => {
    // Prefetch all routes immediately
    import('./pages/GettingStarted');
    import('./pages/APIReference');
    import('./pages/Examples');
    import('./pages/Changelog');
  }, []);

  return <RouterProvider router={router} />;
}
```

**Browser DevTools - Network Tab**:
```
Timeline:
0.0s: Page loads
0.0s: Initial bundle loads (187 KB)
0.7s: Initial bundle finishes
0.7s: Prefetch requests start (all at once)
0.7s: GettingStarted-b1c2d3e4.js (46 KB)
0.7s: APIReference-c2d3e4f5.js (523 KB)
0.7s: Examples-d3e4f5g6.js (313 KB)
0.7s: Changelog-e4f5g6h7.js (79 KB)
3.9s: All prefetch requests complete
```

**Browser Console**:
```
[Network] Downloading 961 KB of prefetch data
[Performance] Main thread blocked for 1.2s parsing prefetched code
[Memory] Heap size increased by 45 MB
```

### Diagnostic Analysis: The Over-Prefetching Failure

**Browser Behavior**:
User loads the home page. It appears quickly (0.7s). But then the browser starts downloading 961 KB of additional JavaScript. The page feels sluggish. Scrolling is janky. Interactions are delayed.

**Network Tab Evidence**:
- 4 simultaneous prefetch requests
- Total: 961 KB (328 KB gzipped)
- On 3G: 3.2 seconds additional download time
- Blocks bandwidth for other resources (images, fonts)

**Performance Metrics**:
- Time to Interactive: 3.9s (worse than no prefetching!)
- Main thread blocked: 1.2s (parsing prefetched code)
- Memory usage: +45 MB
- User experience: Janky, slow

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Fast initial load, instant navigation
   - Actual: Fast initial load, then sluggish page, then fast navigation

2. **What the Network tab reveals**:
   - Key indicator: 961 KB downloaded immediately
   - Bandwidth saturated with prefetch requests
   - Other resources (images, fonts) delayed

3. **Root cause identified**: Prefetching everything defeats the purpose of code splitting. We split the code to reduce initial load, then immediately download it all anyway. The user pays the cost upfront, making the initial experience worse.

4. **Why this approach fails**: Prefetching has costs:
   - Bandwidth (downloads data user may not need)
   - CPU (parsing and compiling prefetched code)
   - Memory (storing prefetched modules)
   - Battery (mobile devices)

5. **What we need**: Strategic prefetching. Prefetch only what's likely to be used, when the browser has spare resources.

### The Goldilocks Principle: Strategic Prefetching

**Good prefetching strategy**:
1. Prefetch high-probability routes (> 50% of users visit)
2. Prefetch on user intent (hover, viewport visibility)
3. Prefetch during idle time (after initial load completes)
4. Respect user preferences (prefers-reduced-data, save-data)
5. Monitor and adjust based on analytics

**Our final, balanced approach**:

```tsx
// src/hooks/useStrategicPrefetch.ts
import { useEffect, useRef } from 'react';

interface PrefetchStrategy {
  route: string;
  priority: 'high' | 'medium' | 'low';
  condition: 'idle' | 'hover' | 'viewport' | 'immediate';
  probability?: number; // 0-1, likelihood user will visit
}

const strategies: PrefetchStrategy[] = [
  {
    route: '/getting-started',
    priority: 'high',
    condition: 'immediate', // Most users visit this
    probability: 0.85
  },
  {
    route: '/api',
    priority: 'medium',
    condition: 'hover', // Large chunk, prefetch on intent
    probability: 0.45
  },
  {
    route: '/examples',
    priority: 'medium',
    condition: 'hover',
    probability: 0.40
  },
  {
    route: '/changelog',
    priority: 'low',
    condition: 'idle', // Rarely visited, prefetch when idle
    probability: 0.15
  }
];

export function useStrategicPrefetch() {
  const prefetchedRoutes = useRef(new Set<string>());

  useEffect(() => {
    // Check if user prefers reduced data
    const prefersReducedData = 
      'connection' in navigator &&
      (navigator as any).connection?.saveData === true;

    if (prefersReducedData) {
      // Don't prefetch on metered connections
      return;
    }

    // Prefetch immediate priority routes
    strategies
      .filter(s => s.condition === 'immediate')
      .forEach(strategy => {
        prefetchRoute(strategy.route);
      });

    // Prefetch idle priority routes when browser is idle
    const idleCallback = 'requestIdleCallback' in window
      ? window.requestIdleCallback
      : (cb: () => void) => setTimeout(cb, 1000);

    const handle = idleCallback(() => {
      strategies
        .filter(s => s.condition === 'idle')
        .forEach(strategy => {
          prefetchRoute(strategy.route);
        });
    });

    return () => {
      if ('cancelIdleCallback' in window) {
        window.cancelIdleCallback(handle as number);
      } else {
        clearTimeout(handle as number);
      }
    };
  }, []);

  const prefetchRoute = (route: string) => {
    if (prefetchedRoutes.current.has(route)) return;

    const routeMap: Record<string, () => Promise<any>> = {
      '/getting-started': () => import('../pages/GettingStarted'),
      '/api': () => import('../pages/APIReference'),
      '/examples': () => import('../pages/Examples'),
      '/changelog': () => import('../pages/Changelog')
    };

    const prefetchFn = routeMap[route];
    if (prefetchFn) {
      prefetchFn()
        .then(() => {
          prefetchedRoutes.current.add(route);
        })
        .catch(() => {
          // Silently fail
        });
    }
  };
}
```

```tsx
// src/App.tsx - Final: Strategic Prefetching
import { useStrategicPrefetch } from './hooks/useStrategicPrefetch';

function App() {
  useStrategicPrefetch();

  return <RouterProvider router={router} />;
}
```

**Browser DevTools - Network Tab** (with strategic prefetching):
```
Timeline:
0.0s: Page loads
0.7s: Initial bundle finishes
0.7s: GettingStarted-b1c2d3e4.js prefetches (immediate, high probability)
1.2s: GettingStarted finishes
2.0s: Browser idle, Changelog-e4f5g6h7.js prefetches (low priority)
2.5s: Changelog finishes
[User hovers over "API Reference"]
3.0s: APIReference-c2d3e4f5.js prefetches (on hover)
```

**Expected vs. Actual Improvement**:

| Metric | No Prefetch | Prefetch All | Strategic Prefetch |
|--------|-------------|--------------|-------------------|
| Initial load | 0.7s | 0.7s | 0.7s |
| Time to Interactive | 0.9s | 3.9s | 0.9s |
| Bandwidth used (first 5s) | 187 KB | 1,148 KB | 233 KB |
| Navigation to /getting-started | 0.8s | 0.1s | 0.1s |
| Navigation to /api (after hover) | 2.1s | 0.1s | 0.2s |
| Navigation to /changelog | 1.5s | 0.1s | 0.1s |

### When to Apply Prefetching

**What it optimizes for**:
- Perceived performance (navigation feels instant)
- User experience on fast connections
- Frequently accessed routes

**What it sacrifices**:
- Bandwidth (may download unused code)
- Battery life (mobile devices)
- Initial page performance (if too aggressive)

**When to prefetch**:

**Immediate prefetch**:
- Routes with > 70% visit probability
- Small chunks (< 50 KB)
- Critical user flows (onboarding, checkout)

**Hover prefetch**:
- Routes with 30-70% visit probability
- Medium chunks (50-200 KB)
- Desktop users (mouse interaction)

**Idle prefetch**:
- Routes with < 30% visit probability
- Large chunks (> 200 KB)
- After initial page load completes

**Viewport prefetch**:
- Links below the fold
- Footer navigation
- Mobile layouts

**When NOT to prefetch**:
- User on metered connection (save-data header)
- User on slow connection (< 2G)
- Routes with < 10% visit probability
- Very large chunks (> 500 KB)
- Routes with personalized content (may be stale)

**Code characteristics**:
- Setup complexity: Medium (need analytics to determine probabilities)
- Maintenance burden: Medium (adjust based on user behavior)
- Performance impact: High (when done right), Negative (when done wrong)

### Respecting User Preferences

Always check for user preferences before prefetching:

```typescript
// src/utils/shouldPrefetch.ts
export function shouldPrefetch(): boolean {
  // Check for Save-Data header
  if ('connection' in navigator) {
    const connection = (navigator as any).connection;
    if (connection?.saveData === true) {
      return false;
    }

    // Check for slow connection
    const effectiveType = connection?.effectiveType;
    if (effectiveType === 'slow-2g' || effectiveType === '2g') {
      return false;
    }
  }

  // Check for reduced motion preference (may indicate low-power mode)
  const prefersReducedMotion = window.matchMedia(
    '(prefers-reduced-motion: reduce)'
  ).matches;
  
  if (prefersReducedMotion) {
    return false;
  }

  // Check battery status (if available)
  if ('getBattery' in navigator) {
    (navigator as any).getBattery().then((battery: any) => {
      if (battery.level < 0.2 || battery.charging === false) {
        return false;
      }
    });
  }

  return true;
}
```

**Ethical prefetching**: Respect user constraints. Don't waste bandwidth on metered connections. Don't drain battery on mobile devices. Prefetching is an optimization, not a requirement.

## The Journey: From Basic Routing to Optimized Navigation

## The Complete Journey

Let's trace the evolution of our documentation site from basic routing to a fully optimized navigation experience:

### The Journey: From Problem to Solution

| Iteration | Problem | Technique Applied | Result | Performance Impact |
|-----------|---------|-------------------|--------|-------------------|
| 0 | All code loads upfront | None | 1.25 MB initial bundle | 2.8s load on 3G |
| 1 | Bundle too large | Route-based code splitting | 187 KB initial bundle | 0.7s load on 3G (75% faster) |
| 2 | Data fetching waterfall | Loader functions | Parallel loading | 1.9s to content (41% faster) |
| 3 | Lost scroll position | ScrollRestoration | Position restored | Better UX |
| 4 | Poor keyboard navigation | Focus management | Focus moves to content | Accessible |
| 5 | Slow navigation on 3G | Strategic prefetching | Instant navigation | Feels immediate |

### Final Implementation

Here's our complete, production-ready routing setup:

```tsx
// src/App.tsx - Production-Ready Routing
import { 
  createBrowserRouter, 
  RouterProvider,
  ScrollRestoration
} from 'react-router-dom';
import { lazy, Suspense } from 'react';
import Layout from './components/Layout';
import RouteAnnouncer from './components/RouteAnnouncer';
import ErrorBoundary from './components/ErrorBoundary';
import { useStrategicPrefetch } from './hooks/useStrategicPrefetch';

// Loaders
import { apiReferenceLoader } from './loaders/apiReferenceLoader';

// Critical path: load immediately
import Home from './pages/Home';
import GettingStarted from './pages/GettingStarted';

// Heavy routes: lazy load
const APIReference = lazy(() => import('./pages/APIReference'));
const Examples = lazy(() => import('./pages/Examples'));
const Changelog = lazy(() => import('./pages/Changelog'));

function RouteLoadingFallback() {
  return (
    <div className="route-loading">
      <div className="spinner" />
      <p>Loading page...</p>
    </div>
  );
}

const router = createBrowserRouter([
  {
    path: '/',
    element: (
      <>
        <ScrollRestoration 
          getKey={(location) => {
            // Restore scroll position for API Reference
            if (location.pathname === '/api') {
              return location.pathname;
            }
            return location.key;
          }}
        />
        <RouteAnnouncer />
        <Layout />
      </>
    ),
    errorElement: <ErrorBoundary />,
    children: [
      {
        index: true,
        element: <Home />
      },
      {
        path: 'getting-started',
        element: <GettingStarted />
      },
      {
        path: 'api',
        element: (
          <Suspense fallback={<RouteLoadingFallback />}>
            <APIReference />
          </Suspense>
        ),
        loader: apiReferenceLoader
      },
      {
        path: 'examples',
        element: (
          <Suspense fallback={<RouteLoadingFallback />}>
            <Examples />
          </Suspense>
        )
      },
      {
        path: 'changelog',
        element: (
          <Suspense fallback={<RouteLoadingFallback />}>
            <Changelog />
          </Suspense>
        )
      }
    ]
  }
]);

function App() {
  useStrategicPrefetch();
  return <RouterProvider router={router} />;
}

export default App;
```

```tsx
// src/components/Layout.tsx - Production-Ready Layout
import { Outlet, useLocation, useNavigation } from 'react-router-dom';
import { useEffect, useRef } from 'react';
import Navigation from './Navigation';

export default function Layout() {
  const location = useLocation();
  const navigation = useNavigation();
  const mainRef = useRef<HTMLElement>(null);
  const isLoading = navigation.state === 'loading';

  // Focus management
  useEffect(() => {
    if (mainRef.current) {
      mainRef.current.focus();
    }
  }, [location.pathname]);

  return (
    <div className="layout">
      {/* Skip link for keyboard users */}
      <a 
        href="#main-content" 
        className="skip-link"
        onClick={(e) => {
          e.preventDefault();
          mainRef.current?.focus();
        }}
      >
        Skip to main content
      </a>

      <Navigation />
      
      {/* Loading indicator */}
      {isLoading && (
        <div className="loading-bar">
          <div className="loading-bar-progress" />
        </div>
      )}
      
      <main 
        id="main-content"
        ref={mainRef}
        tabIndex={-1}
        className="main-content"
      >
        <Outlet />
      </main>
    </div>
  );
}
```

### Decision Framework: Routing Optimization Strategies

When building a React application with routing, use this framework to decide which optimizations to apply:

#### Code Splitting Decision Tree

```
Is the route > 50 KB?
├─ Yes → Split it
│  └─ Is it visited by > 70% of users?
│     ├─ Yes → Keep in initial bundle OR prefetch immediately
│     └─ No → Lazy load
└─ No → Keep in initial bundle
```

#### Data Loading Decision Tree

```
Does the route need data?
├─ Yes → Use loader function
│  └─ Is the data slow to load (> 500ms)?
│     ├─ Yes → Show loading state + consider prefetching
│     └─ No → Loader is sufficient
└─ No → No loader needed
```

#### Prefetching Decision Tree

```
What's the visit probability?
├─ > 70% → Prefetch immediately (or include in initial bundle)
├─ 30-70% → Prefetch on hover/intent
├─ 10-30% → Prefetch on idle
└─ < 10% → Don't prefetch

AND

Is the user on a fast connection?
├─ Yes → Prefetch according to probability
└─ No → Only prefetch high-probability routes (> 70%)

AND

Does the user prefer reduced data?
├─ Yes → Don't prefetch
└─ No → Prefetch according to strategy
```

### Performance Metrics: Before and After

**Initial State** (no optimizations):
- Initial bundle: 1.25 MB (412 KB gzipped)
- Time to Interactive: 3.2s on 3G
- Navigation time: 0s (all code loaded)
- Lighthouse Score: 72
- Accessibility Score: 78

**Final State** (all optimizations):
- Initial bundle: 187 KB (64 KB gzipped)
- Time to Interactive: 0.9s on 3G
- Navigation time: 0.2s average (with prefetching)
- Lighthouse Score: 94
- Accessibility Score: 95

**Improvements**:
- 85% smaller initial bundle
- 72% faster Time to Interactive
- 22 point Lighthouse improvement
- 17 point Accessibility improvement
- Navigation feels instant for most routes

### Lessons Learned

#### 1. Code Splitting is Essential, But Strategic

Don't split everything. Split large, rarely-used routes. Keep small, frequently-used code together. The goal is to optimize initial load without making navigation slow.

#### 2. Data Loading Waterfalls Kill Performance

Use loader functions to fetch data in parallel with component code. Don't wait for the component to mount before fetching data.

#### 3. Accessibility is Not Optional

Scroll restoration and focus management are not "nice to have" features. They're essential for keyboard and screen reader users. Implement them from the start.

#### 4. Prefetching is Powerful, But Respect User Constraints

Prefetch strategically based on visit probability and user intent. Always check for metered connections and user preferences. Don't waste bandwidth or battery.

#### 5. Measure, Don't Guess

Use analytics to determine which routes to prefetch. Use React DevTools Profiler to find performance bottlenecks. Use Lighthouse to validate improvements. Data beats intuition.

### Common Failure Modes and Their Signatures

#### Symptom: Navigation feels slow despite code splitting

**Browser behavior**: User clicks link, waits 2+ seconds, content appears

**Console pattern**: No errors, just slow network requests

**DevTools clues**:
- Network tab shows large chunks downloading
- No prefetching happening
- Sequential requests (waterfall)

**Root cause**: No prefetching strategy, or chunks are too large

**Solution**: Implement strategic prefetching based on user intent and visit probability

#### Symptom: Initial page load is slow despite code splitting

**Browser behavior**: Blank screen for 2+ seconds on initial load

**Console pattern**: Multiple chunk requests immediately after initial bundle

**DevTools clues**:
- Network tab shows many simultaneous requests
- Coverage tab shows low code usage
- Performance tab shows long parse time

**Root cause**: Over-aggressive prefetching or too many immediate imports

**Solution**: Reduce prefetching, increase code splitting granularity, defer non-critical code

#### Symptom: User loses scroll position on back navigation

**Browser behavior**: User navigates back, page scrolls to top

**Console pattern**: No errors

**DevTools clues**:
- React Router navigation events fire
- Scroll position resets to 0

**Root cause**: Missing ScrollRestoration component

**Solution**: Add `<ScrollRestoration />` to router configuration

#### Symptom: Keyboard users can't navigate efficiently

**Browser behavior**: Focus stays on navigation link after route change

**Console pattern**: No errors

**DevTools clues**:
- Active element doesn't change on navigation
- Screen reader doesn't announce page change

**Root cause**: Missing focus management

**Solution**: Implement focus management with `useEffect` and `ref.current.focus()`

### When to Apply These Patterns

**Always apply**:
- Route-based code splitting (for routes > 50 KB)
- Scroll restoration
- Focus management
- Error boundaries

**Apply when appropriate**:
- Loader functions (when routes need data)
- Prefetching (based on visit probability and user constraints)
- Component-level code splitting (for heavy components)

**Rarely apply**:
- Aggressive prefetching (only for high-probability routes)
- Complex prefetching strategies (only when analytics justify it)

### The Professional React Developer's Routing Checklist

Before deploying a React application with routing:

✅ **Code Splitting**
- [ ] Routes > 50 KB are lazy loaded
- [ ] Initial bundle < 200 KB (gzipped)
- [ ] Suspense boundaries provide loading feedback

✅ **Data Loading**
- [ ] Loader functions fetch data in parallel with code
- [ ] Loading states are visible to users
- [ ] Error boundaries catch loader failures

✅ **Accessibility**
- [ ] ScrollRestoration restores scroll position
- [ ] Focus moves to main content on navigation
- [ ] Skip link allows keyboard users to bypass navigation
- [ ] Route changes are announced to screen readers

✅ **Performance**
- [ ] Prefetching strategy based on analytics
- [ ] User preferences respected (save-data, slow connection)
- [ ] Lighthouse score > 90
- [ ] Time to Interactive < 2s on 3G

✅ **User Experience**
- [ ] Navigation feels instant (< 300ms perceived delay)
- [ ] Loading states are clear and consistent
- [ ] Error states are user-friendly
- [ ] Back/forward navigation works correctly

This is the foundation of professional React routing. Master these patterns, and your applications will feel fast, accessible, and polished.
