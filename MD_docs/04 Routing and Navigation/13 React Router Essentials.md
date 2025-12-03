# Chapter 13: React Router Essentials

## Client-side routing basics

## The Problem: Full Page Reloads Break User Experience

Before we dive into React Router, let's understand what problem it solves by building a multi-page application the traditional way—with plain HTML links.

We'll build a simple documentation site with three pages: Home, Docs, and About. This will be our **reference implementation** that we'll progressively enhance throughout this chapter.

**Project Structure**:
```
src/
├── App.tsx          ← Traditional multi-page approach
├── pages/
│   ├── Home.tsx
│   ├── Docs.tsx
│   └── About.tsx
└── main.tsx
```

Let's start with the naive approach: regular HTML links between pages.

```tsx
// src/pages/Home.tsx
export function Home() {
  console.log('Home component rendered');
  
  return (
    <div>
      <h1>Documentation Home</h1>
      <p>Welcome to our documentation site.</p>
      <nav>
        <a href="/docs">View Docs</a> | <a href="/about">About Us</a>
      </nav>
    </div>
  );
}
```

```tsx
// src/pages/Docs.tsx
export function Docs() {
  console.log('Docs component rendered');
  
  return (
    <div>
      <h1>Documentation</h1>
      <p>Here's how to use our product...</p>
      <nav>
        <a href="/">Home</a> | <a href="/about">About Us</a>
      </nav>
    </div>
  );
}
```

```tsx
// src/pages/About.tsx
export function About() {
  console.log('About component rendered');
  
  return (
    <div>
      <h1>About Us</h1>
      <p>We build great documentation tools.</p>
      <nav>
        <a href="/">Home</a> | <a href="/docs">View Docs</a>
      </nav>
    </div>
  );
}
```

```tsx
// src/App.tsx - Version 1: Traditional approach
import { Home } from './pages/Home';
import { Docs } from './pages/Docs';
import { About } from './pages/About';

export function App() {
  // Determine which page to show based on URL
  const path = window.location.pathname;
  
  if (path === '/docs') return <Docs />;
  if (path === '/about') return <About />;
  return <Home />;
}
```

### Diagnostic Analysis: Reading the Failure

Let's run this application and click between pages.

**Browser Behavior**:
- Click "View Docs" link
- Entire page flashes white
- Browser shows loading spinner
- New page appears after ~200-500ms delay
- Any component state is completely lost
- Scroll position resets to top

**Browser Console Output**:
```
Home component rendered
[Page reload - all logs cleared]
Docs component rendered
[Page reload - all logs cleared]
Home component rendered
```

**Network Tab Analysis**:
- Filter: All
- Observation: Every navigation triggers:
  - Request to `/docs` or `/about` (404 error in dev server)
  - Full reload of `main.js` bundle (~500KB)
  - Re-fetch of all CSS files
  - Re-execution of all JavaScript
- Total data transferred per navigation: ~500KB
- Time to interactive: 200-500ms per click

**React DevTools Evidence**:
- Components tab: Entire component tree unmounts and remounts
- Profiler: Cannot measure across navigations (profiler resets)
- No component state persists between navigations

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Instant navigation like a native app
   - Actual: Visible page reload, white flash, loading delay

2. **What the console reveals**:
   - Key indicator: Console clears between navigations
   - This means: JavaScript execution environment is completely reset
   - Root cause: Browser treats each click as a new page request

3. **What the Network tab shows**:
   - Every navigation re-downloads the entire application
   - The browser doesn't know these are "pages" within the same app
   - We're throwing away all loaded JavaScript and starting over

4. **Root cause identified**: 
   HTML `<a>` tags trigger full browser navigation, causing complete page reloads.

5. **Why the current approach can't solve this**:
   Traditional HTML links are designed for multi-page applications where each page is a separate HTML document. React is a single-page application (SPA) framework—the entire app is one HTML document with JavaScript managing what content to display.

6. **What we need**:
   A way to change the URL and displayed content without triggering a browser page reload. This is called **client-side routing**.

### The Concept: Client-Side Routing

Before we introduce React Router, let's understand the mechanism.

**Traditional (Server-Side) Routing**:
1. User clicks `<a href="/docs">`
2. Browser sends HTTP request to server for `/docs`
3. Server responds with new HTML document
4. Browser discards current page and renders new HTML
5. All JavaScript re-executes from scratch

**Client-Side Routing**:
1. User clicks link
2. JavaScript intercepts the click (prevents default browser behavior)
3. JavaScript updates the URL using the History API
4. JavaScript decides which React component to render
5. React updates the DOM (no page reload)

The key insight: **The URL is just another piece of state**. When it changes, React re-renders with different components, but the JavaScript environment stays alive.

### Installing React Router

React Router is the de facto standard for client-side routing in React applications.

```bash
npm install react-router-dom
```

### Iteration 1: Basic Client-Side Routing

Let's refactor our application to use React Router. We'll see the dramatic difference in user experience.

```tsx
// src/App.tsx - Version 2: With React Router
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';
import { Home } from './pages/Home';
import { Docs } from './pages/Docs';
import { About } from './pages/About';

export function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/docs" element={<Docs />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </BrowserRouter>
  );
}
```

```tsx
// src/pages/Home.tsx - Updated with Link components
import { Link } from 'react-router-dom';

export function Home() {
  console.log('Home component rendered');
  
  return (
    <div>
      <h1>Documentation Home</h1>
      <p>Welcome to our documentation site.</p>
      <nav>
        <Link to="/docs">View Docs</Link> | <Link to="/about">About Us</Link>
      </nav>
    </div>
  );
}
```

```tsx
// src/pages/Docs.tsx - Updated with Link components
import { Link } from 'react-router-dom';

export function Docs() {
  console.log('Docs component rendered');
  
  return (
    <div>
      <h1>Documentation</h1>
      <p>Here's how to use our product...</p>
      <nav>
        <Link to="/">Home</Link> | <Link to="/about">About Us</Link>
      </nav>
    </div>
  );
}
```

```tsx
// src/pages/About.tsx - Updated with Link components
import { Link } from 'react-router-dom';

export function About() {
  console.log('About component rendered');
  
  return (
    <div>
      <h1>About Us</h1>
      <p>We build great documentation tools.</p>
      <nav>
        <Link to="/">Home</Link> | <Link to="/docs">View Docs</Link>
      </nav>
    </div>
  );
}
```

### Verification: The Transformation

Run the application again and click between pages.

**Browser Behavior**:
- Click "View Docs" link
- Content changes **instantly** (no white flash)
- No loading spinner
- URL updates in address bar
- Browser back/forward buttons work correctly

**Browser Console Output**:
```
Home component rendered
Docs component rendered
About component rendered
Home component rendered
```

Notice: **No console clearing**. The JavaScript environment stays alive.

**Network Tab Analysis**:
- Filter: All
- Observation: After initial page load, **zero** network requests on navigation
- No re-downloading of JavaScript bundles
- No re-fetching of CSS
- Total data transferred per navigation: **0 bytes**

**React DevTools Evidence**:
- Components tab: Only the page component unmounts/mounts
- `<BrowserRouter>` and `<Routes>` remain mounted
- Profiler: Can measure across navigations
- Component state in other parts of the tree would persist

**Expected vs. Actual Improvement**:
- Navigation time: 200-500ms → **<16ms** (instant)
- Data transferred: 500KB → **0 bytes**
- User experience: Page reload → **Seamless transition**
- State preservation: Lost → **Maintained** (for components outside Routes)

### How It Works: The Machinery Beneath

Let's expose what React Router is doing under the hood.

```tsx
// Conceptual implementation (simplified)
function BrowserRouter({ children }) {
  const [location, setLocation] = useState(window.location.pathname);
  
  useEffect(() => {
    // Listen for browser back/forward button
    const handlePopState = () => {
      setLocation(window.location.pathname);
    };
    
    window.addEventListener('popstate', handlePopState);
    return () => window.removeEventListener('popstate', handlePopState);
  }, []);
  
  // Provide location and navigation function to children
  return (
    <RouterContext.Provider value={{ location, navigate: (path) => {
      window.history.pushState({}, '', path);
      setLocation(path);
    }}}>
      {children}
    </RouterContext.Provider>
  );
}

function Link({ to, children }) {
  const { navigate } = useContext(RouterContext);
  
  return (
    <a 
      href={to}
      onClick={(e) => {
        e.preventDefault(); // Stop browser navigation
        navigate(to);       // Update URL and trigger re-render
      }}
    >
      {children}
    </a>
  );
}
```

**Key mechanisms**:

1. **`<BrowserRouter>`** wraps your app and manages URL state
2. **`<Link>`** components intercept clicks and prevent default browser behavior
3. **History API** (`pushState`) updates the URL without page reload
4. **Context** provides current location to all components
5. **`<Routes>`** matches current URL to the correct component

### When to Apply This Solution

**What it optimizes for**:
- Instant navigation (no page reloads)
- Preserved JavaScript state
- Native app-like user experience
- Reduced bandwidth usage

**What it sacrifices**:
- Initial bundle size (includes routing logic)
- SEO complexity (requires server-side rendering or pre-rendering)
- Browser compatibility (requires History API support)

**When to choose this approach**:
- Building a web application (not a content site)
- User will navigate frequently between pages
- You need to preserve state across navigation
- You're already using React

**When to avoid this approach**:
- Building a static content site (use Next.js or Astro instead)
- SEO is critical and you can't do SSR
- Target browsers don't support History API (IE9 and below)

**Code characteristics**:
- Setup complexity: Low (wrap app in `<BrowserRouter>`)
- Maintenance burden: Low (declarative route definitions)
- Performance impact: Positive (eliminates page reloads)

### Limitation Preview

This solves instant navigation, but we still have issues:

1. **No shared layout**: Navigation is duplicated in every page component
2. **No 404 handling**: Invalid URLs show blank page
3. **No programmatic navigation**: Can't navigate from button clicks or after form submission
4. **No URL parameters**: Can't build routes like `/docs/:id`

We'll address these in the following sections.

## Routes, links, and navigation

## Building a Proper Navigation System

Our current implementation has navigation links duplicated in every page component. Let's fix this by introducing a shared layout and exploring all the ways to navigate in React Router.

### Iteration 2: Shared Layout with Navigation

**Current state recap**: Our app has client-side routing, but each page duplicates the navigation menu.

**Current limitation**: Changing the navigation requires updating three separate files.

**New scenario introduction**: What happens when we want to add a fourth page or change the navigation structure?

```tsx
// src/components/Layout.tsx - New file
import { Link, Outlet } from 'react-router-dom';

export function Layout() {
  return (
    <div className="app-layout">
      <header>
        <nav className="main-nav">
          <Link to="/">Home</Link>
          <Link to="/docs">Docs</Link>
          <Link to="/about">About</Link>
        </nav>
      </header>
      
      <main>
        <Outlet /> {/* Child routes render here */}
      </main>
      
      <footer>
        <p>© 2025 Documentation Site</p>
      </footer>
    </div>
  );
}
```

```tsx
// src/App.tsx - Version 3: With shared layout
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { Layout } from './components/Layout';
import { Home } from './pages/Home';
import { Docs } from './pages/Docs';
import { About } from './pages/About';

export function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route index element={<Home />} />
          <Route path="docs" element={<Docs />} />
          <Route path="about" element={<About />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

```tsx
// src/pages/Home.tsx - Version 3: Simplified (no navigation)
export function Home() {
  return (
    <div>
      <h1>Documentation Home</h1>
      <p>Welcome to our documentation site.</p>
    </div>
  );
}
```

```tsx
// src/pages/Docs.tsx - Version 3: Simplified
export function Docs() {
  return (
    <div>
      <h1>Documentation</h1>
      <p>Here's how to use our product...</p>
    </div>
  );
}
```

```tsx
// src/pages/About.tsx - Version 3: Simplified
export function About() {
  return (
    <div>
      <h1>About Us</h1>
      <p>We build great documentation tools.</p>
    </div>
  );
}
```

**Improvement**: Navigation is now defined once in `Layout.tsx`. Page components focus only on their content.

**How `<Outlet>` works**:
- `<Layout>` renders the shared structure (header, nav, footer)
- `<Outlet />` is a placeholder that renders the matched child route
- When URL is `/docs`, `<Outlet />` renders `<Docs />`
- When URL is `/about`, `<Outlet />` renders `<About />`

**Note on `index` route**:
- `<Route index element={<Home />} />` matches the parent path exactly (`/`)
- It's equivalent to `<Route path="" element={<Home />} />` but more explicit

### Active Link Styling

Users need visual feedback showing which page they're on. React Router provides `NavLink` for this.

```tsx
// src/components/Layout.tsx - Version 2: With active link styling
import { NavLink, Outlet } from 'react-router-dom';
import './Layout.css';

export function Layout() {
  return (
    <div className="app-layout">
      <header>
        <nav className="main-nav">
          <NavLink 
            to="/" 
            className={({ isActive }) => isActive ? 'nav-link active' : 'nav-link'}
          >
            Home
          </NavLink>
          <NavLink 
            to="/docs"
            className={({ isActive }) => isActive ? 'nav-link active' : 'nav-link'}
          >
            Docs
          </NavLink>
          <NavLink 
            to="/about"
            className={({ isActive }) => isActive ? 'nav-link active' : 'nav-link'}
          >
            About
          </NavLink>
        </nav>
      </header>
      
      <main>
        <Outlet />
      </main>
      
      <footer>
        <p>© 2025 Documentation Site</p>
      </footer>
    </div>
  );
}
```

```css
/* src/components/Layout.css */
.main-nav {
  display: flex;
  gap: 1rem;
  padding: 1rem;
  background: #f5f5f5;
}

.nav-link {
  padding: 0.5rem 1rem;
  text-decoration: none;
  color: #333;
  border-radius: 4px;
  transition: background 0.2s;
}

.nav-link:hover {
  background: #e0e0e0;
}

.nav-link.active {
  background: #007bff;
  color: white;
  font-weight: bold;
}
```

**`NavLink` vs. `Link`**:
- `NavLink` receives `isActive` prop indicating if it matches current URL
- Use `NavLink` for navigation menus where you need active state
- Use `Link` for inline links in content where active state doesn't matter

### Programmatic Navigation

Sometimes you need to navigate from JavaScript code, not just from link clicks. Common scenarios:
- After form submission
- After successful login
- Redirecting based on conditions
- Navigating from button clicks

React Router provides the `useNavigate` hook for this.

```tsx
// src/pages/Docs.tsx - Version 4: With programmatic navigation
import { useNavigate } from 'react-router-dom';
import { useState } from 'react';

export function Docs() {
  const navigate = useNavigate();
  const [searchQuery, setSearchQuery] = useState('');
  
  const handleSearch = (e: React.FormEvent) => {
    e.preventDefault();
    
    if (searchQuery.trim()) {
      // Navigate to search results page (we'll build this later)
      navigate(`/docs/search?q=${encodeURIComponent(searchQuery)}`);
    }
  };
  
  const handleBackToHome = () => {
    // Navigate back to home
    navigate('/');
  };
  
  return (
    <div>
      <h1>Documentation</h1>
      
      <form onSubmit={handleSearch}>
        <input
          type="text"
          value={searchQuery}
          onChange={(e) => setSearchQuery(e.target.value)}
          placeholder="Search docs..."
        />
        <button type="submit">Search</button>
      </form>
      
      <p>Here's how to use our product...</p>
      
      <button onClick={handleBackToHome}>
        Back to Home
      </button>
    </div>
  );
}
```

**`navigate()` options**:

```typescript
// Navigate to a path
navigate('/about');

// Navigate with relative path
navigate('../'); // Go up one level
navigate('settings'); // Go to sibling route

// Navigate with state (accessible in destination component)
navigate('/docs', { state: { fromSearch: true } });

// Replace current history entry (back button won't return here)
navigate('/login', { replace: true });

// Navigate backwards/forwards in history
navigate(-1); // Go back
navigate(-2); // Go back twice
navigate(1);  // Go forward
```

### The Failure: 404 Pages

**Current limitation**: If a user navigates to an invalid URL like `/invalid-page`, they see a blank screen.

**New scenario introduction**: What happens when someone types a wrong URL or follows a broken link?

Let's test this:
1. Navigate to `http://localhost:5173/nonexistent`
2. Observe the result

**Browser Behavior**:
- Blank page (only header and footer visible)
- No error message
- No indication that something went wrong

**Browser Console Output**:
```
No routes matched location "/nonexistent"
```

**React DevTools Evidence**:
- Components tab: `<Layout>` is rendered
- `<Outlet />` renders nothing (no matched route)
- No error boundary triggered

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Error message or redirect to home
   - Actual: Confusing blank page

2. **What the console reveals**:
   - Key indicator: "No routes matched location"
   - This means: React Router found no matching route definition

3. **Root cause identified**: 
   We haven't defined a catch-all route for unmatched URLs.

4. **What we need**: 
   A 404 page that renders when no other routes match.

### Solution: Catch-All Route

```tsx
// src/pages/NotFound.tsx - New file
import { Link } from 'react-router-dom';

export function NotFound() {
  return (
    <div style={{ textAlign: 'center', padding: '2rem' }}>
      <h1>404 - Page Not Found</h1>
      <p>The page you're looking for doesn't exist.</p>
      <Link to="/">Go back home</Link>
    </div>
  );
}
```

```tsx
// src/App.tsx - Version 4: With 404 handling
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { Layout } from './components/Layout';
import { Home } from './pages/Home';
import { Docs } from './pages/Docs';
import { About } from './pages/About';
import { NotFound } from './pages/NotFound';

export function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route index element={<Home />} />
          <Route path="docs" element={<Docs />} />
          <Route path="about" element={<About />} />
          <Route path="*" element={<NotFound />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

**Verification**: Navigate to `/nonexistent` again.

**Browser Behavior**:
- 404 page renders with clear error message
- User can click link to return home
- URL stays at `/nonexistent` (doesn't redirect)

**The `*` wildcard**:
- Matches any path that wasn't matched by previous routes
- Must be defined last (routes are matched in order)
- Can be used at any nesting level

### Navigation with State

Sometimes you need to pass data between routes without putting it in the URL.

```tsx
// src/pages/Home.tsx - Version 4: Passing state on navigation
import { useNavigate } from 'react-router-dom';

export function Home() {
  const navigate = useNavigate();
  
  const handleGetStarted = () => {
    navigate('/docs', { 
      state: { 
        fromHome: true,
        timestamp: Date.now() 
      } 
    });
  };
  
  return (
    <div>
      <h1>Documentation Home</h1>
      <p>Welcome to our documentation site.</p>
      <button onClick={handleGetStarted}>Get Started</button>
    </div>
  );
}
```

```tsx
// src/pages/Docs.tsx - Version 5: Receiving navigation state
import { useLocation, useNavigate } from 'react-router-dom';

export function Docs() {
  const location = useLocation();
  const navigate = useNavigate();
  
  // Access state passed from previous navigation
  const fromHome = location.state?.fromHome;
  const timestamp = location.state?.timestamp;
  
  return (
    <div>
      <h1>Documentation</h1>
      
      {fromHome && (
        <div style={{ background: '#e3f2fd', padding: '1rem', marginBottom: '1rem' }}>
          Welcome! You navigated here from the home page at {new Date(timestamp).toLocaleTimeString()}.
        </div>
      )}
      
      <p>Here's how to use our product...</p>
      
      <button onClick={() => navigate('/')}>
        Back to Home
      </button>
    </div>
  );
}
```

**When to use navigation state**:
- Passing temporary data that shouldn't be in URL
- Showing success messages after form submission
- Preserving scroll position or UI state
- Passing data that's too large for URL parameters

**When NOT to use navigation state**:
- Data that should be shareable via URL
- Data that should persist on page refresh
- Data that should be bookmarkable

**Important**: Navigation state is lost on page refresh. For persistent data, use URL parameters (next section) or localStorage.

### When to Apply This Solution

**What it optimizes for**:
- Centralized navigation management
- Visual feedback for current page
- Flexible navigation patterns (links, buttons, programmatic)
- Graceful handling of invalid URLs

**What it sacrifices**:
- Slightly more complex component structure (layouts and outlets)
- Need to understand routing concepts (nesting, outlets, wildcards)

**When to choose this approach**:
- Building any multi-page React application
- Need consistent navigation across pages
- Want to handle 404s gracefully
- Need programmatic navigation

**Code characteristics**:
- Setup complexity: Medium (requires understanding layouts and outlets)
- Maintenance burden: Low (centralized navigation)
- Performance impact: Minimal (no additional re-renders)

### Limitation Preview

This solves shared layouts and navigation, but we still need:

1. **Dynamic routes**: URLs like `/docs/getting-started` or `/docs/api-reference`
2. **URL parameters**: Extracting values from URLs like `/user/:id`
3. **Query strings**: Handling search parameters like `?q=search&page=2`

We'll address these in the next section.

## URL parameters and query strings

## Dynamic Routes and URL Data

So far, our routes have been static: `/`, `/docs`, `/about`. Real applications need dynamic routes that respond to URL parameters—like `/docs/getting-started` or `/user/123`.

### Iteration 3: Dynamic Documentation Pages

**Current state recap**: Our app has a single `/docs` page with static content.

**Current limitation**: We can't have separate pages for different documentation topics.

**New scenario introduction**: What if we want URLs like:
- `/docs/getting-started`
- `/docs/api-reference`
- `/docs/deployment`

Let's build a dynamic documentation system.

**Updated Project Structure**:
```
src/
├── components/
│   └── Layout.tsx
├── pages/
│   ├── Home.tsx
│   ├── Docs.tsx          ← Will become docs index
│   ├── DocPage.tsx       ← New: Individual doc pages
│   ├── About.tsx
│   └── NotFound.tsx
└── data/
    └── docs.ts           ← New: Documentation content
```

```typescript
// src/data/docs.ts - Documentation content database
export interface DocContent {
  id: string;
  title: string;
  content: string;
  category: string;
}

export const docsDatabase: DocContent[] = [
  {
    id: 'getting-started',
    title: 'Getting Started',
    content: 'Learn how to set up and use our product...',
    category: 'Basics'
  },
  {
    id: 'api-reference',
    title: 'API Reference',
    content: 'Complete API documentation...',
    category: 'Reference'
  },
  {
    id: 'deployment',
    title: 'Deployment Guide',
    content: 'How to deploy your application...',
    category: 'Advanced'
  },
  {
    id: 'troubleshooting',
    title: 'Troubleshooting',
    content: 'Common issues and solutions...',
    category: 'Support'
  }
];

export function getDocById(id: string): DocContent | undefined {
  return docsDatabase.find(doc => doc.id === id);
}

export function getAllDocs(): DocContent[] {
  return docsDatabase;
}
```

```tsx
// src/pages/Docs.tsx - Version 6: Documentation index page
import { Link } from 'react-router-dom';
import { getAllDocs } from '../data/docs';

export function Docs() {
  const docs = getAllDocs();
  
  // Group docs by category
  const categories = docs.reduce((acc, doc) => {
    if (!acc[doc.category]) {
      acc[doc.category] = [];
    }
    acc[doc.category].push(doc);
    return acc;
  }, {} as Record<string, typeof docs>);
  
  return (
    <div>
      <h1>Documentation</h1>
      <p>Browse our documentation by category:</p>
      
      {Object.entries(categories).map(([category, categoryDocs]) => (
        <div key={category} style={{ marginBottom: '2rem' }}>
          <h2>{category}</h2>
          <ul>
            {categoryDocs.map(doc => (
              <li key={doc.id}>
                <Link to={`/docs/${doc.id}`}>{doc.title}</Link>
              </li>
            ))}
          </ul>
        </div>
      ))}
    </div>
  );
}
```

```tsx
// src/pages/DocPage.tsx - New file: Individual documentation page
import { useParams, Link, Navigate } from 'react-router-dom';
import { getDocById } from '../data/docs';

export function DocPage() {
  // Extract the 'id' parameter from the URL
  const { id } = useParams<{ id: string }>();
  
  // Fetch the documentation content
  const doc = id ? getDocById(id) : undefined;
  
  // If doc doesn't exist, redirect to 404
  if (!doc) {
    return <Navigate to="/404" replace />;
  }
  
  return (
    <div>
      <nav style={{ marginBottom: '1rem' }}>
        <Link to="/docs">← Back to all docs</Link>
      </nav>
      
      <article>
        <h1>{doc.title}</h1>
        <p style={{ color: '#666', fontSize: '0.9rem' }}>
          Category: {doc.category}
        </p>
        <div style={{ marginTop: '2rem' }}>
          {doc.content}
        </div>
      </article>
    </div>
  );
}
```

```tsx
// src/App.tsx - Version 5: With dynamic routes
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { Layout } from './components/Layout';
import { Home } from './pages/Home';
import { Docs } from './pages/Docs';
import { DocPage } from './pages/DocPage';
import { About } from './pages/About';
import { NotFound } from './pages/NotFound';

export function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route index element={<Home />} />
          <Route path="docs">
            <Route index element={<Docs />} />
            <Route path=":id" element={<DocPage />} />
          </Route>
          <Route path="about" element={<About />} />
          <Route path="*" element={<NotFound />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

**Verification**: 
1. Navigate to `/docs` - see the documentation index
2. Click "Getting Started" - URL becomes `/docs/getting-started`
3. See the individual doc page render
4. Click "Back to all docs" - return to index

**Browser Console Output**:
```
(No errors - smooth navigation)
```

**React DevTools Evidence**:
- Components tab: `<DocPage>` component renders
- Props: `params: { id: "getting-started" }`
- When navigating between docs, only `<DocPage>` re-renders
- `<Layout>` and `<Docs>` (when on index) remain mounted

**How URL parameters work**:

1. **Route definition**: `<Route path=":id" element={<DocPage />} />`
   - `:id` is a parameter placeholder
   - Matches any value in that URL segment
   - The parameter name (`id`) is arbitrary—you choose it

2. **Extracting parameters**: `const { id } = useParams<{ id: string }>()`
   - `useParams()` hook returns an object with all URL parameters
   - TypeScript generic provides type safety
   - Parameters are always strings (even if they look like numbers)

3. **Matching behavior**:
   - `/docs/getting-started` → `{ id: "getting-started" }`
   - `/docs/api-reference` → `{ id: "api-reference" }`
   - `/docs/123` → `{ id: "123" }`
   - `/docs` → No match (index route renders instead)

### The Failure: Invalid URL Parameters

**Current limitation**: What happens if someone navigates to `/docs/nonexistent-page`?

Let's test this:
1. Navigate to `/docs/invalid-doc-id`
2. Observe the result

**Browser Behavior**:
- Page redirects to `/404`
- 404 page renders
- URL changes to `/404`

**Browser Console Output**:
```
(No errors - redirect happens cleanly)
```

**This is actually correct behavior!** We handled it with:

```tsx
// In DocPage.tsx
if (!doc) {
  return <Navigate to="/404" replace />;
}
```

**`<Navigate>` component**:
- Renders a redirect instead of content
- `replace` prop replaces current history entry (back button skips this URL)
- Alternative to `navigate()` hook when redirect is based on render logic

### Multiple URL Parameters

Routes can have multiple parameters.

```tsx
// Example: Blog post with category and slug
<Route path="blog/:category/:slug" element={<BlogPost />} />

// In component:
function BlogPost() {
  const { category, slug } = useParams<{ category: string; slug: string }>();
  
  // URL: /blog/tutorials/react-hooks
  // category = "tutorials"
  // slug = "react-hooks"
  
  return <div>Category: {category}, Slug: {slug}</div>;
}
```

### Optional Parameters

Sometimes parameters should be optional.

```tsx
// Optional parameter with '?'
<Route path="users/:id?" element={<UserList />} />

// In component:
function UserList() {
  const { id } = useParams<{ id?: string }>();
  
  if (id) {
    // Show specific user
    return <UserDetail userId={id} />;
  }
  
  // Show all users
  return <AllUsers />;
}
```

### Query Strings: Search Parameters

URL parameters (`:id`) are for required, structural data. Query strings (`?key=value`) are for optional, filtering data.

**Example URLs**:
- `/docs?search=hooks` - Search query
- `/docs?category=basics&sort=date` - Multiple filters
- `/docs/api-reference?version=2.0` - Version selector

Let's add search functionality to our docs.

```tsx
// src/pages/Docs.tsx - Version 7: With search functionality
import { Link, useSearchParams } from 'react-router-dom';
import { getAllDocs } from '../data/docs';
import { useState, useEffect } from 'react';

export function Docs() {
  const [searchParams, setSearchParams] = useSearchParams();
  const [searchQuery, setSearchQuery] = useState(searchParams.get('q') || '');
  
  const docs = getAllDocs();
  
  // Filter docs based on search query
  const filteredDocs = searchQuery
    ? docs.filter(doc => 
        doc.title.toLowerCase().includes(searchQuery.toLowerCase()) ||
        doc.content.toLowerCase().includes(searchQuery.toLowerCase())
      )
    : docs;
  
  // Group filtered docs by category
  const categories = filteredDocs.reduce((acc, doc) => {
    if (!acc[doc.category]) {
      acc[doc.category] = [];
    }
    acc[doc.category].push(doc);
    return acc;
  }, {} as Record<string, typeof docs>);
  
  const handleSearch = (e: React.FormEvent) => {
    e.preventDefault();
    
    if (searchQuery.trim()) {
      // Update URL with search query
      setSearchParams({ q: searchQuery });
    } else {
      // Remove search query from URL
      setSearchParams({});
    }
  };
  
  const handleClearSearch = () => {
    setSearchQuery('');
    setSearchParams({});
  };
  
  return (
    <div>
      <h1>Documentation</h1>
      
      <form onSubmit={handleSearch} style={{ marginBottom: '2rem' }}>
        <input
          type="text"
          value={searchQuery}
          onChange={(e) => setSearchQuery(e.target.value)}
          placeholder="Search documentation..."
          style={{ padding: '0.5rem', width: '300px' }}
        />
        <button type="submit" style={{ marginLeft: '0.5rem' }}>
          Search
        </button>
        {searchParams.get('q') && (
          <button 
            type="button" 
            onClick={handleClearSearch}
            style={{ marginLeft: '0.5rem' }}
          >
            Clear
          </button>
        )}
      </form>
      
      {searchParams.get('q') && (
        <p style={{ color: '#666', marginBottom: '1rem' }}>
          Found {filteredDocs.length} result(s) for "{searchParams.get('q')}"
        </p>
      )}
      
      {Object.keys(categories).length === 0 ? (
        <p>No documentation found matching your search.</p>
      ) : (
        Object.entries(categories).map(([category, categoryDocs]) => (
          <div key={category} style={{ marginBottom: '2rem' }}>
            <h2>{category}</h2>
            <ul>
              {categoryDocs.map(doc => (
                <li key={doc.id}>
                  <Link to={`/docs/${doc.id}`}>{doc.title}</Link>
                </li>
              ))}
            </ul>
          </div>
        ))
      )}
    </div>
  );
}
```

**Verification**:
1. Navigate to `/docs`
2. Type "api" in search box and submit
3. URL becomes `/docs?q=api`
4. See filtered results
5. Click "Clear" - URL becomes `/docs` again
6. Results show all docs

**Browser Behavior**:
- Search query appears in URL
- URL is shareable (copy/paste preserves search)
- Browser back/forward buttons work with search history
- Page refresh preserves search query

**`useSearchParams()` hook**:

```typescript
const [searchParams, setSearchParams] = useSearchParams();

// Reading query parameters
const query = searchParams.get('q');           // Single value
const page = searchParams.get('page');         // Another parameter
const tags = searchParams.getAll('tag');       // Multiple values with same key

// Setting query parameters
setSearchParams({ q: 'hooks' });               // ?q=hooks
setSearchParams({ q: 'hooks', page: '2' });    // ?q=hooks&page=2

// Updating existing parameters
setSearchParams(prev => {
  prev.set('page', '3');                       // Update one parameter
  return prev;
});

// Removing parameters
setSearchParams(prev => {
  prev.delete('q');                            // Remove one parameter
  return prev;
});

// Clear all parameters
setSearchParams({});
```

### Reading Query Parameters Without State

Sometimes you just need to read query parameters without managing them as state.

```tsx
import { useSearchParams } from 'react-router-dom';

function DocPage() {
  const [searchParams] = useSearchParams();
  
  // Read parameters directly
  const version = searchParams.get('version') || '1.0';
  const highlight = searchParams.get('highlight');
  
  return (
    <div>
      <p>Viewing version: {version}</p>
      {highlight && <p>Highlighting: {highlight}</p>}
    </div>
  );
}
```

### Combining URL Parameters and Query Strings

```tsx
// Route with both parameter and query string
<Route path="docs/:id" element={<DocPage />} />

// Component using both
function DocPage() {
  const { id } = useParams<{ id: string }>();
  const [searchParams] = useSearchParams();
  
  const version = searchParams.get('version');
  const highlight = searchParams.get('highlight');
  
  // URL: /docs/api-reference?version=2.0&highlight=useState
  // id = "api-reference"
  // version = "2.0"
  // highlight = "useState"
  
  return (
    <div>
      <h1>Doc: {id}</h1>
      {version && <p>Version: {version}</p>}
      {highlight && <p>Highlighting: {highlight}</p>}
    </div>
  );
}
```

### When to Apply This Solution

**URL Parameters (`:id`)**:

**What it optimizes for**:
- Required, structural data (user IDs, slugs, categories)
- Clean, readable URLs
- SEO-friendly paths

**When to use**:
- Data that defines the resource being viewed
- Data that should be in the URL path
- Required navigation data

**When to avoid**:
- Optional filtering or sorting
- Temporary UI state
- Data that changes frequently

**Query Strings (`?key=value`)**:

**What it optimizes for**:
- Optional filtering, sorting, pagination
- Shareable search results
- Preserving UI state in URL

**When to use**:
- Search queries
- Filters and sorting options
- Pagination state
- Optional view settings

**When to avoid**:
- Required data (use URL parameters)
- Sensitive data (use state or localStorage)
- Very large data (use state management)

**Code characteristics**:
- Setup complexity: Low (built-in hooks)
- Maintenance burden: Low (declarative)
- Performance impact: Minimal (no extra re-renders)

### Limitation Preview

This solves dynamic routes and URL data, but we still need:

1. **Nested routes with shared layouts**: Documentation sections with their own navigation
2. **Route-based code splitting**: Loading page code only when needed
3. **Protected routes**: Restricting access based on authentication

We'll address nested routes and layouts in the next section.

## Nested routes and layouts

## Building Complex Route Hierarchies

Real applications have complex navigation structures. Documentation sites have sections with subsections. Admin panels have nested settings pages. E-commerce sites have category hierarchies.

React Router's nested routes let you build these hierarchies while keeping layouts and navigation DRY (Don't Repeat Yourself).

### Iteration 4: Documentation Sections with Nested Navigation

**Current state recap**: Our docs have individual pages, but no organizational structure beyond categories.

**Current limitation**: All documentation pages share the same layout. We can't have section-specific navigation.

**New scenario introduction**: What if we want:
- A "Guides" section with its own sidebar navigation
- An "API Reference" section with different navigation
- Each section maintains the main site header but has its own sub-navigation

**Updated Project Structure**:
```
src/
├── components/
│   ├── Layout.tsx              ← Main site layout
│   ├── GuidesLayout.tsx        ← New: Guides section layout
│   └── ApiLayout.tsx           ← New: API section layout
├── pages/
│   ├── Home.tsx
│   ├── Docs.tsx
│   ├── guides/                 ← New: Guides section pages
│   │   ├── GettingStarted.tsx
│   │   ├── Installation.tsx
│   │   └── Configuration.tsx
│   ├── api/                    ← New: API section pages
│   │   ├── Overview.tsx
│   │   ├── Authentication.tsx
│   │   └── Endpoints.tsx
│   ├── About.tsx
│   └── NotFound.tsx
└── App.tsx
```

Let's build the guides section first.

```tsx
// src/pages/guides/GettingStarted.tsx - New file
export function GettingStarted() {
  return (
    <article>
      <h1>Getting Started</h1>
      <p>Welcome to our product! This guide will help you get up and running quickly.</p>
      
      <h2>Prerequisites</h2>
      <ul>
        <li>Node.js 18 or higher</li>
        <li>npm or yarn</li>
        <li>Basic JavaScript knowledge</li>
      </ul>
      
      <h2>Quick Start</h2>
      <p>Follow these steps to create your first project...</p>
    </article>
  );
}
```

```tsx
// src/pages/guides/Installation.tsx - New file
export function Installation() {
  return (
    <article>
      <h1>Installation</h1>
      <p>Learn how to install our product in your project.</p>
      
      <h2>Using npm</h2>
      <pre>npm install our-product
```

```
yarn add our-product
```

```tsx
// src/pages/guides/Configuration.tsx - New file
export function Configuration() {
  return (
    <article>
      <h1>Configuration</h1>
      <p>Configure our product to match your needs.</p>
      
      <h2>Basic Configuration</h2>
      <p>Create a config file in your project root...</p>
      
      <h2>Advanced Options</h2>
      <p>For advanced use cases, you can customize...</p>
    </article>
  );
}
```

```tsx
// src/components/GuidesLayout.tsx - New file: Section-specific layout
import { NavLink, Outlet } from 'react-router-dom';

export function GuidesLayout() {
  return (
    <div style={{ display: 'flex', gap: '2rem' }}>
      {/* Sidebar navigation for guides section */}
      <aside style={{ 
        width: '200px', 
        borderRight: '1px solid #ddd',
        paddingRight: '1rem'
      }}>
        <h2 style={{ fontSize: '1.2rem', marginBottom: '1rem' }}>Guides</h2>
        <nav style={{ display: 'flex', flexDirection: 'column', gap: '0.5rem' }}>
          <NavLink 
            to="/guides/getting-started"
            className={({ isActive }) => isActive ? 'active' : ''}
            style={({ isActive }) => ({
              padding: '0.5rem',
              textDecoration: 'none',
              color: isActive ? '#007bff' : '#333',
              background: isActive ? '#e3f2fd' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Getting Started
          </NavLink>
          <NavLink 
            to="/guides/installation"
            className={({ isActive }) => isActive ? 'active' : ''}
            style={({ isActive }) => ({
              padding: '0.5rem',
              textDecoration: 'none',
              color: isActive ? '#007bff' : '#333',
              background: isActive ? '#e3f2fd' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Installation
          </NavLink>
          <NavLink 
            to="/guides/configuration"
            className={({ isActive }) => isActive ? 'active' : ''}
            style={({ isActive }) => ({
              padding: '0.5rem',
              textDecoration: 'none',
              color: isActive ? '#007bff' : '#333',
              background: isActive ? '#e3f2fd' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Configuration
          </NavLink>
        </nav>
      </aside>
      
      {/* Content area where child routes render */}
      <main style={{ flex: 1 }}>
        <Outlet />
      </main>
    </div>
  );
}
```

```tsx
// src/App.tsx - Version 6: With nested routes
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { Layout } from './components/Layout';
import { GuidesLayout } from './components/GuidesLayout';
import { Home } from './pages/Home';
import { Docs } from './pages/Docs';
import { GettingStarted } from './pages/guides/GettingStarted';
import { Installation } from './pages/guides/Installation';
import { Configuration } from './pages/guides/Configuration';
import { About } from './pages/About';
import { NotFound } from './pages/NotFound';

export function App() {
  return (
    <BrowserRouter>
      <Routes>
        {/* Main site layout */}
        <Route path="/" element={<Layout />}>
          <Route index element={<Home />} />
          <Route path="docs" element={<Docs />} />
          
          {/* Nested guides section with its own layout */}
          <Route path="guides" element={<GuidesLayout />}>
            <Route index element={<GettingStarted />} />
            <Route path="getting-started" element={<GettingStarted />} />
            <Route path="installation" element={<Installation />} />
            <Route path="configuration" element={<Configuration />} />
          </Route>
          
          <Route path="about" element={<About />} />
          <Route path="*" element={<NotFound />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

```tsx
// src/components/Layout.tsx - Version 3: Updated navigation
import { NavLink, Outlet } from 'react-router-dom';

export function Layout() {
  return (
    <div className="app-layout">
      <header>
        <nav className="main-nav" style={{ 
          display: 'flex', 
          gap: '1rem', 
          padding: '1rem',
          background: '#f5f5f5',
          borderBottom: '2px solid #ddd'
        }}>
          <NavLink 
            to="/"
            style={({ isActive }) => ({
              padding: '0.5rem 1rem',
              textDecoration: 'none',
              color: isActive ? 'white' : '#333',
              background: isActive ? '#007bff' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Home
          </NavLink>
          <NavLink 
            to="/docs"
            style={({ isActive }) => ({
              padding: '0.5rem 1rem',
              textDecoration: 'none',
              color: isActive ? 'white' : '#333',
              background: isActive ? '#007bff' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Docs
          </NavLink>
          <NavLink 
            to="/guides"
            style={({ isActive }) => ({
              padding: '0.5rem 1rem',
              textDecoration: 'none',
              color: isActive ? 'white' : '#333',
              background: isActive ? '#007bff' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Guides
          </NavLink>
          <NavLink 
            to="/about"
            style={({ isActive }) => ({
              padding: '0.5rem 1rem',
              textDecoration: 'none',
              color: isActive ? 'white' : '#333',
              background: isActive ? '#007bff' : 'transparent',
              borderRadius: '4px'
            })}
          >
            About
          </NavLink>
        </nav>
      </header>
      
      <main style={{ padding: '2rem' }}>
        <Outlet />
      </main>
      
      <footer style={{ 
        padding: '1rem', 
        background: '#f5f5f5',
        borderTop: '1px solid #ddd',
        textAlign: 'center'
      }}>
        <p>© 2025 Documentation Site</p>
      </footer>
    </div>
  );
}
```

**Verification**:
1. Navigate to `/guides` - see Getting Started page with sidebar
2. Click "Installation" in sidebar - content changes, sidebar stays
3. Click "Configuration" - content changes again
4. Click "Home" in main nav - sidebar disappears, back to home page

**Browser Behavior**:
- Main header stays visible on all pages
- Guides sidebar only appears on `/guides/*` routes
- Sidebar navigation highlights current page
- Content area updates without sidebar re-rendering

**React DevTools Evidence**:
- Components tab hierarchy:
  ```
  <Layout>
    <header> (stays mounted)
    <main>
      <GuidesLayout>
        <aside> (stays mounted within guides section)
        <main>
          <GettingStarted> (unmounts/mounts on navigation)
        </main>
      </GuidesLayout>
    </main>
    <footer> (stays mounted)
  ```
- When navigating within guides: Only the innermost component changes
- When navigating away from guides: `<GuidesLayout>` unmounts entirely

**How nested routes work**:

1. **Route hierarchy matches URL structure**:
   ```tsx
   <Route path="/" element={<Layout />}>           // Matches: /
     <Route path="guides" element={<GuidesLayout />}>  // Matches: /guides
       <Route path="installation" element={<Installation />} />  // Matches: /guides/installation
     </Route>
   </Route>
   ```

2. **Each level renders its `<Outlet />`**:
   - `<Layout>` renders `<Outlet />` → renders `<GuidesLayout>`
   - `<GuidesLayout>` renders `<Outlet />` → renders `<Installation>`

3. **Layouts persist at their level**:
   - Main header/footer persist across all routes
   - Guides sidebar persists within guides section
   - Only the deepest component changes on navigation

### Adding the API Reference Section

Let's add a second nested section to demonstrate the pattern.

```tsx
// src/pages/api/Overview.tsx - New file
export function Overview() {
  return (
    <article>
      <h1>API Overview</h1>
      <p>Our API provides programmatic access to all product features.</p>
      
      <h2>Base URL</h2>
      <pre>https://api.example.com/v1
```

```tsx
// src/pages/api/Authentication.tsx - New file
export function Authentication() {
  return (
    <article>
      <h1>Authentication</h1>
      <p>Learn how to authenticate your API requests.</p>
      
      <h2>API Keys</h2>
      <p>Generate an API key from your dashboard...</p>
      
      <h2>Using Your Key</h2>
      <pre>Authorization: Bearer YOUR_API_KEY
```

```tsx
// src/pages/api/Endpoints.tsx - New file
export function Endpoints() {
  return (
    <article>
      <h1>API Endpoints</h1>
      <p>Complete reference of all available endpoints.</p>
      
      <h2>Users</h2>
      <ul>
        <li>GET /users
```

```
GET /users/:id
```

```
POST /users
```

```
GET /products
```

```
GET /products/:id
```

```tsx
// src/components/ApiLayout.tsx - New file
import { NavLink, Outlet } from 'react-router-dom';

export function ApiLayout() {
  return (
    <div style={{ display: 'flex', gap: '2rem' }}>
      <aside style={{ 
        width: '200px', 
        borderRight: '1px solid #ddd',
        paddingRight: '1rem'
      }}>
        <h2 style={{ fontSize: '1.2rem', marginBottom: '1rem' }}>API Reference</h2>
        <nav style={{ display: 'flex', flexDirection: 'column', gap: '0.5rem' }}>
          <NavLink 
            to="/api/overview"
            style={({ isActive }) => ({
              padding: '0.5rem',
              textDecoration: 'none',
              color: isActive ? '#007bff' : '#333',
              background: isActive ? '#e3f2fd' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Overview
          </NavLink>
          <NavLink 
            to="/api/authentication"
            style={({ isActive }) => ({
              padding: '0.5rem',
              textDecoration: 'none',
              color: isActive ? '#007bff' : '#333',
              background: isActive ? '#e3f2fd' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Authentication
          </NavLink>
          <NavLink 
            to="/api/endpoints"
            style={({ isActive }) => ({
              padding: '0.5rem',
              textDecoration: 'none',
              color: isActive ? '#007bff' : '#333',
              background: isActive ? '#e3f2fd' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Endpoints
          </NavLink>
        </nav>
      </aside>
      
      <main style={{ flex: 1 }}>
        <Outlet />
      </main>
    </div>
  );
}
```

```tsx
// src/App.tsx - Version 7: With both nested sections
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { Layout } from './components/Layout';
import { GuidesLayout } from './components/GuidesLayout';
import { ApiLayout } from './components/ApiLayout';
import { Home } from './pages/Home';
import { Docs } from './pages/Docs';
import { GettingStarted } from './pages/guides/GettingStarted';
import { Installation } from './pages/guides/Installation';
import { Configuration } from './pages/guides/Configuration';
import { Overview } from './pages/api/Overview';
import { Authentication } from './pages/api/Authentication';
import { Endpoints } from './pages/api/Endpoints';
import { About } from './pages/About';
import { NotFound } from './pages/NotFound';

export function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route index element={<Home />} />
          <Route path="docs" element={<Docs />} />
          
          {/* Guides section */}
          <Route path="guides" element={<GuidesLayout />}>
            <Route index element={<GettingStarted />} />
            <Route path="getting-started" element={<GettingStarted />} />
            <Route path="installation" element={<Installation />} />
            <Route path="configuration" element={<Configuration />} />
          </Route>
          
          {/* API section */}
          <Route path="api" element={<ApiLayout />}>
            <Route index element={<Overview />} />
            <Route path="overview" element={<Overview />} />
            <Route path="authentication" element={<Authentication />} />
            <Route path="endpoints" element={<Endpoints />} />
          </Route>
          
          <Route path="about" element={<About />} />
          <Route path="*" element={<NotFound />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

```tsx
// src/components/Layout.tsx - Version 4: With API link
import { NavLink, Outlet } from 'react-router-dom';

export function Layout() {
  return (
    <div className="app-layout">
      <header>
        <nav className="main-nav" style={{ 
          display: 'flex', 
          gap: '1rem', 
          padding: '1rem',
          background: '#f5f5f5',
          borderBottom: '2px solid #ddd'
        }}>
          <NavLink 
            to="/"
            style={({ isActive }) => ({
              padding: '0.5rem 1rem',
              textDecoration: 'none',
              color: isActive ? 'white' : '#333',
              background: isActive ? '#007bff' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Home
          </NavLink>
          <NavLink 
            to="/docs"
            style={({ isActive }) => ({
              padding: '0.5rem 1rem',
              textDecoration: 'none',
              color: isActive ? 'white' : '#333',
              background: isActive ? '#007bff' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Docs
          </NavLink>
          <NavLink 
            to="/guides"
            style={({ isActive }) => ({
              padding: '0.5rem 1rem',
              textDecoration: 'none',
              color: isActive ? 'white' : '#333',
              background: isActive ? '#007bff' : 'transparent',
              borderRadius: '4px'
            })}
          >
            Guides
          </NavLink>
          <NavLink 
            to="/api"
            style={({ isActive }) => ({
              padding: '0.5rem 1rem',
              textDecoration: 'none',
              color: isActive ? 'white' : '#333',
              background: isActive ? '#007bff' : 'transparent',
              borderRadius: '4px'
            })}
          >
            API
          </NavLink>
          <NavLink 
            to="/about"
            style={({ isActive }) => ({
              padding: '0.5rem 1rem',
              textDecoration: 'none',
              color: isActive ? 'white' : '#333',
              background: isActive ? '#007bff' : 'transparent',
              borderRadius: '4px'
            })}
          >
            About
          </NavLink>
        </nav>
      </header>
      
      <main style={{ padding: '2rem' }}>
        <Outlet />
      </main>
      
      <footer style={{ 
        padding: '1rem', 
        background: '#f5f5f5',
        borderTop: '1px solid #ddd',
        textAlign: 'center'
      }}>
        <p>© 2025 Documentation Site</p>
      </footer>
    </div>
  );
}
```

**Verification**:
1. Navigate between Guides and API sections
2. Notice each section has its own sidebar
3. Main header/footer persist across all sections
4. Sidebar content changes when switching sections

**Component mounting behavior**:
- Navigate from `/guides/installation` to `/api/overview`:
  - `<GuidesLayout>` unmounts (including its sidebar)
  - `<ApiLayout>` mounts (with its sidebar)
  - `<Layout>` stays mounted (header/footer persist)

### Index Routes Explained

You may have noticed this pattern:

```tsx
<Route path="guides" element={<GuidesLayout />}>
  <Route index element={<GettingStarted />} />
  <Route path="getting-started" element={<GettingStarted />} />
  {/* ... */}
</Route>
```

**Why both `index` and `getting-started`?**

- `<Route index />` matches the parent path exactly (`/guides`)
- `<Route path="getting-started" />` matches `/guides/getting-started`
- We use the same component for both to provide a default page

**Alternative approach** (redirect to default):

```tsx
import { Navigate } from 'react-router-dom';

<Route path="guides" element={<GuidesLayout />}>
  <Route index element={<Navigate to="/guides/getting-started" replace />} />
  <Route path="getting-started" element={<GettingStarted />} />
  {/* ... */}
</Route>
```

This redirects `/guides` to `/guides/getting-started` instead of rendering the same component twice.

### Relative Navigation in Nested Routes

When you're deep in a nested route, you can use relative paths.

```tsx
// In /guides/installation component
import { Link } from 'react-router-dom';

function Installation() {
  return (
    <div>
      <h1>Installation</h1>
      
      {/* Relative to current route */}
      <Link to="../configuration">Next: Configuration</Link>
      
      {/* Relative to current route */}
      <Link to="../getting-started">Back: Getting Started</Link>
      
      {/* Absolute path */}
      <Link to="/api/overview">See API Overview</Link>
    </div>
  );
}
```

**Relative path rules**:
- `to="configuration"` → `/guides/configuration` (sibling route)
- `to="../configuration"` → `/guides/configuration` (up one level, then down)
- `to="../../api/overview"` → `/api/overview` (up two levels, then down)
- `to="/api/overview"` → `/api/overview` (absolute path)

### When to Apply This Solution

**What it optimizes for**:
- Section-specific layouts and navigation
- DRY principle (no duplicated navigation code)
- Logical URL structure matching content hierarchy
- Persistent layouts at each nesting level

**What it sacrifices**:
- More complex route configuration
- Need to understand outlet and nesting concepts
- Slightly more components (layout components)

**When to choose this approach**:
- Building documentation sites with sections
- Admin panels with nested settings
- E-commerce sites with category hierarchies
- Any app with distinct sections that share some layout

**When to avoid this approach**:
- Simple sites with flat navigation
- When all pages share identical layout
- When section-specific navigation isn't needed

**Code characteristics**:
- Setup complexity: Medium (requires understanding nesting)
- Maintenance burden: Low (centralized section layouts)
- Performance impact: Positive (layouts don't re-render unnecessarily)

### Limitation Preview

This solves nested layouts and section-specific navigation, but we still need:

1. **Protected routes**: Restricting access based on authentication
2. **Route guards**: Running logic before entering a route
3. **Scroll restoration**: Maintaining scroll position on navigation

We'll address protected routes in the next section.

## Protected routes

## Restricting Access with Authentication

Most applications have pages that should only be accessible to authenticated users. Admin panels, user dashboards, account settings—these all need protection.

Let's build a protected route system that redirects unauthenticated users to a login page.

### Iteration 5: Adding Authentication

**Current state recap**: Our documentation site is completely public. Anyone can access any page.

**Current limitation**: We can't restrict access to certain pages based on user authentication.

**New scenario introduction**: What if we want to add:
- A user dashboard at `/dashboard`
- Account settings at `/settings`
- Both should require login

**Updated Project Structure**:
```
src/
├── components/
│   ├── Layout.tsx
│   ├── GuidesLayout.tsx
│   ├── ApiLayout.tsx
│   └── ProtectedRoute.tsx    ← New: Route protection wrapper
├── pages/
│   ├── Login.tsx              ← New: Login page
│   ├── Dashboard.tsx          ← New: Protected page
│   ├── Settings.tsx           ← New: Protected page
│   └── [existing pages...]
├── context/
│   └── AuthContext.tsx        ← New: Authentication state
└── App.tsx
```

First, let's create a simple authentication context to manage login state.

```tsx
// src/context/AuthContext.tsx - New file
import { createContext, useContext, useState, ReactNode } from 'react';

interface AuthContextType {
  isAuthenticated: boolean;
  user: { id: string; name: string; email: string } | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [user, setUser] = useState<{ id: string; name: string; email: string } | null>(null);
  
  const login = async (email: string, password: string) => {
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    // In real app, validate credentials with backend
    if (email && password) {
      setIsAuthenticated(true);
      setUser({
        id: '1',
        name: 'John Doe',
        email: email
      });
    } else {
      throw new Error('Invalid credentials');
    }
  };
  
  const logout = () => {
    setIsAuthenticated(false);
    setUser(null);
  };
  
  return (
    <AuthContext.Provider value={{ isAuthenticated, user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

```tsx
// src/pages/Login.tsx - New file
import { useState } from 'react';
import { useNavigate, useLocation } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export function Login() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  
  const { login } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();
  
  // Get the page user was trying to access (if any)
  const from = (location.state as any)?.from?.pathname || '/dashboard';
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    setIsLoading(true);
    
    try {
      await login(email, password);
      // Redirect to the page they were trying to access
      navigate(from, { replace: true });
    } catch (err) {
      setError('Invalid email or password');
    } finally {
      setIsLoading(false);
    }
  };
  
  return (
    <div style={{ maxWidth: '400px', margin: '2rem auto', padding: '2rem', border: '1px solid #ddd', borderRadius: '8px' }}>
      <h1>Login</h1>
      
      {error && (
        <div style={{ padding: '1rem', background: '#fee', color: '#c00', borderRadius: '4px', marginBottom: '1rem' }}>
          {error}
        </div>
      )}
      
      <form onSubmit={handleSubmit}>
        <div style={{ marginBottom: '1rem' }}>
          <label style={{ display: 'block', marginBottom: '0.5rem' }}>
            Email:
            <input
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              required
              style={{ display: 'block', width: '100%', padding: '0.5rem', marginTop: '0.25rem' }}
            />
          </label>
        </div>
        
        <div style={{ marginBottom: '1rem' }}>
          <label style={{ display: 'block', marginBottom: '0.5rem' }}>
            Password:
            <input
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              required
              style={{ display: 'block', width: '100%', padding: '0.5rem', marginTop: '0.25rem' }}
            />
          </label>
        </div>
        
        <button 
          type="submit" 
          disabled={isLoading}
          style={{ 
            width: '100%', 
            padding: '0.75rem', 
            background: '#007bff', 
            color: 'white', 
            border: 'none', 
            borderRadius: '4px',
            cursor: isLoading ? 'not-allowed' : 'pointer'
          }}
        >
          {isLoading ? 'Logging in...' : 'Login'}
        </button>
      </form>
      
      <p style={{ marginTop: '1rem', fontSize: '0.9rem', color: '#666' }}>
        Demo: Use any email and password to login
      </p>
    </div>
  );
}
```

```tsx
// src/pages/Dashboard.tsx - New file: Protected page
import { useAuth } from '../context/AuthContext';
import { useNavigate } from 'react-router-dom';

export function Dashboard() {
  const { user, logout } = useAuth();
  const navigate = useNavigate();
  
  const handleLogout = () => {
    logout();
    navigate('/login');
  };
  
  return (
    <div>
      <h1>Dashboard</h1>
      <p>Welcome back, {user?.name}!</p>
      
      <div style={{ marginTop: '2rem', padding: '1rem', background: '#f5f5f5', borderRadius: '4px' }}>
        <h2>Your Account</h2>
        <p><strong>Email:</strong> {user?.email}</p>
        <p><strong>User ID:</strong> {user?.id}</p>
      </div>
      
      <button 
        onClick={handleLogout}
        style={{ 
          marginTop: '2rem',
          padding: '0.5rem 1rem',
          background: '#dc3545',
          color: 'white',
          border: 'none',
          borderRadius: '4px',
          cursor: 'pointer'
        }}
      >
        Logout
      </button>
    </div>
  );
}
```

```tsx
// src/pages/Settings.tsx - New file: Another protected page
import { useAuth } from '../context/AuthContext';
import { useState } from 'react';

export function Settings() {
  const { user } = useAuth();
  const [name, setName] = useState(user?.name || '');
  const [email, setEmail] = useState(user?.email || '');
  
  const handleSave = (e: React.FormEvent) => {
    e.preventDefault();
    // In real app, save to backend
    alert('Settings saved! (Demo only)');
  };
  
  return (
    <div>
      <h1>Account Settings</h1>
      
      <form onSubmit={handleSave} style={{ maxWidth: '500px' }}>
        <div style={{ marginBottom: '1rem' }}>
          <label style={{ display: 'block', marginBottom: '0.5rem' }}>
            Name:
            <input
              type="text"
              value={name}
              onChange={(e) => setName(e.target.value)}
              style={{ display: 'block', width: '100%', padding: '0.5rem', marginTop: '0.25rem' }}
            />
          </label>
        </div>
        
        <div style={{ marginBottom: '1rem' }}>
          <label style={{ display: 'block', marginBottom: '0.5rem' }}>
            Email:
            <input
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              style={{ display: 'block', width: '100%', padding: '0.5rem', marginTop: '0.25rem' }}
            />
          </label>
        </div>
        
        <button 
          type="submit"
          style={{ 
            padding: '0.5rem 1rem',
            background: '#007bff',
            color: 'white',
            border: 'none',
            borderRadius: '4px',
            cursor: 'pointer'
          }}
        >
          Save Changes
        </button>
      </form>
    </div>
  );
}
```

Now let's create the protected route wrapper component.

```tsx
// src/components/ProtectedRoute.tsx - New file
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';
import { ReactNode } from 'react';

interface ProtectedRouteProps {
  children: ReactNode;
}

export function ProtectedRoute({ children }: ProtectedRouteProps) {
  const { isAuthenticated } = useAuth();
  const location = useLocation();
  
  if (!isAuthenticated) {
    // Redirect to login, but save the location they were trying to access
    return <Navigate to="/login" state={{ from: location }} replace />;
  }
  
  return <>{children}</>;
}
```

```tsx
// src/App.tsx - Version 8: With protected routes
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';
import { ProtectedRoute } from './components/ProtectedRoute';
import { Layout } from './components/Layout';
import { GuidesLayout } from './components/GuidesLayout';
import { ApiLayout } from './components/ApiLayout';
import { Home } from './pages/Home';
import { Login } from './pages/Login';
import { Dashboard } from './pages/Dashboard';
import { Settings } from './pages/Settings';
import { Docs } from './pages/Docs';
import { GettingStarted } from './pages/guides/GettingStarted';
import { Installation } from './pages/guides/Installation';
import { Configuration } from './pages/guides/Configuration';
import { Overview } from './pages/api/Overview';
import { Authentication } from './pages/api/Authentication';
import { Endpoints } from './pages/api/Endpoints';
import { About } from './pages/About';
import { NotFound } from './pages/NotFound';

export function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          {/* Public routes */}
          <Route path="/login" element={<Login />} />
          
          {/* Routes with main layout */}
          <Route path="/" element={<Layout />}>
            <Route index element={<Home />} />
            <Route path="docs" element={<Docs />} />
            
            <Route path="guides" element={<GuidesLayout />}>
              <Route index element={<GettingStarted />} />
              <Route path="getting-started" element={<GettingStarted />} />
              <Route path="installation" element={<Installation />} />
              <Route path="configuration" element={<Configuration />} />
            </Route>
            
            <Route path="api" element={<ApiLayout />}>
              <Route index element={<Overview />} />
              <Route path="overview" element={<Overview />} />
              <Route path="authentication" element={<Authentication />} />
              <Route path="endpoints" element={<Endpoints />} />
            </Route>
            
            {/* Protected routes */}
            <Route 
              path="dashboard" 
              element={
                <ProtectedRoute>
                  <Dashboard />
                </ProtectedRoute>
              } 
            />
            <Route 
              path="settings" 
              element={
                <ProtectedRoute>
                  <Settings />
                </ProtectedRoute>
              } 
            />
            
            <Route path="about" element={<About />} />
            <Route path="*" element={<NotFound />} />
          </Route>
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}
```

```tsx
// src/components/Layout.tsx - Version 5: With auth-aware navigation
import { NavLink, Outlet, Link } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export function Layout() {
  const { isAuthenticated, user, logout } = useAuth();
  
  return (
    <div className="app-layout">
      <header>
        <nav className="main-nav" style={{ 
          display: 'flex', 
          gap: '1rem', 
          padding: '1rem',
          background: '#f5f5f5',
          borderBottom: '2px solid #ddd',
          alignItems: 'center'
        }}>
          <div style={{ display: 'flex', gap: '1rem', flex: 1 }}>
            <NavLink 
              to="/"
              style={({ isActive }) => ({
                padding: '0.5rem 1rem',
                textDecoration: 'none',
                color: isActive ? 'white' : '#333',
                background: isActive ? '#007bff' : 'transparent',
                borderRadius: '4px'
              })}
            >
              Home
            </NavLink>
            <NavLink 
              to="/docs"
              style={({ isActive }) => ({
                padding: '0.5rem 1rem',
                textDecoration: 'none',
                color: isActive ? 'white' : '#333',
                background: isActive ? '#007bff' : 'transparent',
                borderRadius: '4px'
              })}
            >
              Docs
            </NavLink>
            <NavLink 
              to="/guides"
              style={({ isActive }) => ({
                padding: '0.5rem 1rem',
                textDecoration: 'none',
                color: isActive ? 'white' : '#333',
                background: isActive ? '#007bff' : 'transparent',
                borderRadius: '4px'
              })}
            >
              Guides
            </NavLink>
            <NavLink 
              to="/api"
              style={({ isActive }) => ({
                padding: '0.5rem 1rem',
                textDecoration: 'none',
                color: isActive ? 'white' : '#333',
                background: isActive ? '#007bff' : 'transparent',
                borderRadius: '4px'
              })}
            >
              API
            </NavLink>
            
            {isAuthenticated && (
              <>
                <NavLink 
                  to="/dashboard"
                  style={({ isActive }) => ({
                    padding: '0.5rem 1rem',
                    textDecoration: 'none',
                    color: isActive ? 'white' : '#333',
                    background: isActive ? '#007bff' : 'transparent',
                    borderRadius: '4px'
                  })}
                >
                  Dashboard
                </NavLink>
                <NavLink 
                  to="/settings"
                  style={({ isActive }) => ({
                    padding: '0.5rem 1rem',
                    textDecoration: 'none',
                    color: isActive ? 'white' : '#333',
                    background: isActive ? '#007bff' : 'transparent',
                    borderRadius: '4px'
                  })}
                >
                  Settings
                </NavLink>
              </>
            )}
          </div>
          
          <div style={{ display: 'flex', gap: '1rem', alignItems: 'center' }}>
            {isAuthenticated ? (
              <>
                <span style={{ color: '#666' }}>
                  {user?.name}
                </span>
                <button
                  onClick={logout}
                  style={{
                    padding: '0.5rem 1rem',
                    background: '#dc3545',
                    color: 'white',
                    border: 'none',
                    borderRadius: '4px',
                    cursor: 'pointer'
                  }}
                >
                  Logout
                </button>
              </>
            ) : (
              <Link 
                to="/login"
                style={{
                  padding: '0.5rem 1rem',
                  textDecoration: 'none',
                  color: 'white',
                  background: '#28a745',
                  borderRadius: '4px'
                }}
              >
                Login
              </Link>
            )}
          </div>
        </nav>
      </header>
      
      <main style={{ padding: '2rem' }}>
        <Outlet />
      </main>
      
      <footer style={{ 
        padding: '1rem', 
        background: '#f5f5f5',
        borderTop: '1px solid #ddd',
        textAlign: 'center'
      }}>
        <p>© 2025 Documentation Site</p>
      </footer>
    </div>
  );
}
```

### Diagnostic Analysis: Testing Protection

Let's test the protected route system.

**Test 1: Accessing protected route while logged out**

1. Navigate to `/dashboard` without logging in
2. Observe the result

**Browser Behavior**:
- Immediately redirects to `/login`
- URL changes from `/dashboard` to `/login`
- Login form appears

**Browser Console Output**:
```
(No errors - clean redirect)
```

**React DevTools Evidence**:
- Components tab: `<Navigate>` component renders briefly
- Then `<Login>` component mounts
- `location.state` contains: `{ from: { pathname: "/dashboard" } }`

**Test 2: Logging in and accessing protected route**

1. Enter any email and password in login form
2. Submit form
3. Observe the result

**Browser Behavior**:
- Form shows "Logging in..." for 1 second (simulated API call)
- Redirects to `/dashboard` (the page we were trying to access)
- Dashboard shows user information
- Navigation shows "Dashboard" and "Settings" links
- User name appears in header

**Browser Console Output**:
```
(No errors - successful authentication flow)
```

**Test 3: Accessing protected route while logged in**

1. While logged in, navigate to `/settings`
2. Observe the result

**Browser Behavior**:
- Settings page loads immediately
- No redirect to login
- User can modify settings

**Test 4: Logging out**

1. Click "Logout" button in header
2. Try to navigate to `/dashboard`
3. Observe the result

**Browser Behavior**:
- User is logged out
- Dashboard and Settings links disappear from navigation
- Attempting to access `/dashboard` redirects to `/login`

**How protected routes work**:

1. **Wrapper component checks authentication**:
   ```tsx
   if (!isAuthenticated) {
     return <Navigate to="/login" state={{ from: location }} replace />;
   }
   ```

2. **Saves attempted destination**:
   - `state={{ from: location }}` stores where user was trying to go
   - Login component reads this: `const from = location.state?.from?.pathname`

3. **Redirects after successful login**:
   - `navigate(from, { replace: true })` sends user to original destination
   - `replace: true` removes login page from history (back button skips it)

### Role-Based Access Control

Sometimes you need more granular control than just "logged in" or "logged out". Let's add role-based protection.

```tsx
// src/context/AuthContext.tsx - Version 2: With roles
import { createContext, useContext, useState, ReactNode } from 'react';

type UserRole = 'user' | 'admin';

interface User {
  id: string;
  name: string;
  email: string;
  role: UserRole;
}

interface AuthContextType {
  isAuthenticated: boolean;
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  hasRole: (role: UserRole) => boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [user, setUser] = useState<User | null>(null);
  
  const login = async (email: string, password: string) => {
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    if (email && password) {
      // Simulate different roles based on email
      const role: UserRole = email.includes('admin') ? 'admin' : 'user';
      
      setIsAuthenticated(true);
      setUser({
        id: '1',
        name: email.includes('admin') ? 'Admin User' : 'Regular User',
        email: email,
        role: role
      });
    } else {
      throw new Error('Invalid credentials');
    }
  };
  
  const logout = () => {
    setIsAuthenticated(false);
    setUser(null);
  };
  
  const hasRole = (role: UserRole) => {
    return user?.role === role;
  };
  
  return (
    <AuthContext.Provider value={{ isAuthenticated, user, login, logout, hasRole }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

```tsx
// src/components/ProtectedRoute.tsx - Version 2: With role checking
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';
import { ReactNode } from 'react';

type UserRole = 'user' | 'admin';

interface ProtectedRouteProps {
  children: ReactNode;
  requireRole?: UserRole;
}

export function ProtectedRoute({ children, requireRole }: ProtectedRouteProps) {
  const { isAuthenticated, hasRole } = useAuth();
  const location = useLocation();
  
  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }
  
  if (requireRole && !hasRole(requireRole)) {
    // User is authenticated but doesn't have required role
    return (
      <div style={{ padding: '2rem', textAlign: 'center' }}>
        <h1>Access Denied</h1>
        <p>You don't have permission to access this page.</p>
        <p>Required role: {requireRole}</p>
      </div>
    );
  }
  
  return <>{children}</>;
}
```

```tsx
// src/pages/AdminPanel.tsx - New file: Admin-only page
import { useAuth } from '../context/AuthContext';

export function AdminPanel() {
  const { user } = useAuth();
  
  return (
    <div>
      <h1>Admin Panel</h1>
      <p>Welcome, {user?.name}. You have administrative access.</p>
      
      <div style={{ marginTop: '2rem' }}>
        <h2>Admin Functions</h2>
        <ul>
          <li>Manage users</li>
          <li>View system logs</li>
          <li>Configure settings</li>
          <li>Generate reports</li>
        </ul>
      </div>
      
      <div style={{ 
        marginTop: '2rem', 
        padding: '1rem', 
        background: '#fff3cd', 
        border: '1px solid #ffc107',
        borderRadius: '4px'
      }}>
        <strong>Note:</strong> This page is only accessible to users with admin role.
      </div>
    </div>
  );
}
```

```tsx
// src/App.tsx - Version 9: With role-based routes
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';
import { ProtectedRoute } from './components/ProtectedRoute';
import { Layout } from './components/Layout';
import { GuidesLayout } from './components/GuidesLayout';
import { ApiLayout } from './components/ApiLayout';
import { Home } from './pages/Home';
import { Login } from './pages/Login';
import { Dashboard } from './pages/Dashboard';
import { Settings } from './pages/Settings';
import { AdminPanel } from './pages/AdminPanel';
import { Docs } from './pages/Docs';
import { GettingStarted } from './pages/guides/GettingStarted';
import { Installation } from './pages/guides/Installation';
import { Configuration } from './pages/guides/Configuration';
import { Overview } from './pages/api/Overview';
import { Authentication } from './pages/api/Authentication';
import { Endpoints } from './pages/api/Endpoints';
import { About } from './pages/About';
import { NotFound } from './pages/NotFound';

export function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          <Route path="/login" element={<Login />} />
          
          <Route path="/" element={<Layout />}>
            <Route index element={<Home />} />
            <Route path="docs" element={<Docs />} />
            
            <Route path="guides" element={<GuidesLayout />}>
              <Route index element={<GettingStarted />} />
              <Route path="getting-started" element={<GettingStarted />} />
              <Route path="installation" element={<Installation />} />
              <Route path="configuration" element={<Configuration />} />
            </Route>
            
            <Route path="api" element={<ApiLayout />}>
              <Route index element={<Overview />} />
              <Route path="overview" element={<Overview />} />
              <Route path="authentication" element={<Authentication />} />
              <Route path="endpoints" element={<Endpoints />} />
            </Route>
            
            {/* Protected routes - any authenticated user */}
            <Route 
              path="dashboard" 
              element={
                <ProtectedRoute>
                  <Dashboard />
                </ProtectedRoute>
              } 
            />
            <Route 
              path="settings" 
              element={
                <ProtectedRoute>
                  <Settings />
                </ProtectedRoute>
              } 
            />
            
            {/* Admin-only route */}
            <Route 
              path="admin" 
              element={
                <ProtectedRoute requireRole="admin">
                  <AdminPanel />
                </ProtectedRoute>
              } 
            />
            
            <Route path="about" element={<About />} />
            <Route path="*" element={<NotFound />} />
          </Route>
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}
```

```tsx
// src/components/Layout.tsx - Version 6: With admin link
import { NavLink, Outlet, Link } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export function Layout() {
  const { isAuthenticated, user, logout, hasRole } = useAuth();
  
  return (
    <div className="app-layout">
      <header>
        <nav className="main-nav" style={{ 
          display: 'flex', 
          gap: '1rem', 
          padding: '1rem',
          background: '#f5f5f5',
          borderBottom: '2px solid #ddd',
          alignItems: 'center'
        }}>
          <div style={{ display: 'flex', gap: '1rem', flex: 1 }}>
            <NavLink 
              to="/"
              style={({ isActive }) => ({
                padding: '0.5rem 1rem',
                textDecoration: 'none',
                color: isActive ? 'white' : '#333',
                background: isActive ? '#007bff' : 'transparent',
                borderRadius: '4px'
              })}
            >
              Home
            </NavLink>
            <NavLink 
              to="/docs"
              style={({ isActive }) => ({
                padding: '0.5rem 1rem',
                textDecoration: 'none',
                color: isActive ? 'white' : '#333',
                background: isActive ? '#007bff' : 'transparent',
                borderRadius: '4px'
              })}
            >
              Docs
            </NavLink>
            <NavLink 
              to="/guides"
              style={({ isActive }) => ({
                padding: '0.5rem 1rem',
                textDecoration: 'none',
                color: isActive ? 'white' : '#333',
                background: isActive ? '#007bff' : 'transparent',
                borderRadius: '4px'
              })}
            >
              Guides
            </NavLink>
            <NavLink 
              to="/api"
              style={({ isActive }) => ({
                padding: '0.5rem 1rem',
                textDecoration: 'none',
                color: isActive ? 'white' : '#333',
                background: isActive ? '#007bff' : 'transparent',
                borderRadius: '4px'
              })}
            >
              API
            </NavLink>
            
            {isAuthenticated && (
              <>
                <NavLink 
                  to="/dashboard"
                  style={({ isActive }) => ({
                    padding: '0.5rem 1rem',
                    textDecoration: 'none',
                    color: isActive ? 'white' : '#333',
                    background: isActive ? '#007bff' : 'transparent',
                    borderRadius: '4px'
                  })}
                >
                  Dashboard
                </NavLink>
                <NavLink 
                  to="/settings"
                  style={({ isActive }) => ({
                    padding: '0.5rem 1rem',
                    textDecoration: 'none',
                    color: isActive ? 'white' : '#333',
                    background: isActive ? '#007bff' : 'transparent',
                    borderRadius: '4px'
                  })}
                >
                  Settings
                </NavLink>
                
                {hasRole('admin') && (
                  <NavLink 
                    to="/admin"
                    style={({ isActive }) => ({
                      padding: '0.5rem 1rem',
                      textDecoration: 'none',
                      color: isActive ? 'white' : '#333',
                      background: isActive ? '#dc3545' : 'transparent',
                      borderRadius: '4px'
                    })}
                  >
                    Admin
                  </NavLink>
                )}
              </>
            )}
          </div>
          
          <div style={{ display: 'flex', gap: '1rem', alignItems: 'center' }}>
            {isAuthenticated ? (
              <>
                <span style={{ color: '#666' }}>
                  {user?.name} ({user?.role})
                </span>
                <button
                  onClick={logout}
                  style={{
                    padding: '0.5rem 1rem',
                    background: '#dc3545',
                    color: 'white',
                    border: 'none',
                    borderRadius: '4px',
                    cursor: 'pointer'
                  }}
                >
                  Logout
                </button>
              </>
            ) : (
              <Link 
                to="/login"
                style={{
                  padding: '0.5rem 1rem',
                  textDecoration: 'none',
                  color: 'white',
                  background: '#28a745',
                  borderRadius: '4px'
                }}
              >
                Login
              </Link>
            )}
          </div>
        </nav>
      </header>
      
      <main style={{ padding: '2rem' }}>
        <Outlet />
      </main>
      
      <footer style={{ 
        padding: '1rem', 
        background: '#f5f5f5',
        borderTop: '1px solid #ddd',
        textAlign: 'center'
      }}>
        <p>© 2025 Documentation Site</p>
      </footer>
    </div>
  );
}
```

**Verification**:

**Test 1: Regular user**
1. Login with `user@example.com`
2. See Dashboard and Settings links (no Admin link)
3. Try to navigate to `/admin` directly
4. See "Access Denied" message

**Test 2: Admin user**
1. Logout and login with `admin@example.com`
2. See Dashboard, Settings, and Admin links
3. Navigate to `/admin`
4. See admin panel content

**Browser Behavior**:
- Regular users: Admin link hidden, direct access shows error
- Admin users: Admin link visible, full access granted
- Role displayed in header: "Regular User (user)" or "Admin User (admin)"

### When to Apply This Solution

**What it optimizes for**:
- Secure access control
- User experience (redirect to login, then back to destination)
- Role-based permissions
- Conditional UI rendering based on auth state

**What it sacrifices**:
- Additional context and wrapper components
- Need to manage authentication state
- Slightly more complex route configuration

**When to choose this approach**:
- Building apps with user accounts
- Need to restrict access to certain pages
- Different user roles with different permissions
- Want to preserve intended destination after login

**When to avoid this approach**:
- Completely public applications
- When backend handles all authorization
- Simple apps where all pages are accessible to all users

**Code characteristics**:
- Setup complexity: Medium (requires auth context and wrapper)
- Maintenance burden: Low (centralized protection logic)
- Performance impact: Minimal (one extra component in tree)
- Security: Client-side only (always validate on backend too!)

**Important security note**: Client-side route protection is for **user experience**, not security. Always validate permissions on your backend. A determined user can bypass client-side checks by modifying JavaScript. Protected routes prevent accidental access and provide good UX, but your API must enforce authorization.

## The Journey: From Problem to Solution

## The Complete Journey: From Traditional Navigation to Modern Routing

Let's review how our documentation site evolved from a traditional multi-page application to a sophisticated single-page application with client-side routing.

### The Journey: From Problem to Solution

| Iteration | Failure Mode | Technique Applied | Result | Performance Impact |
|-----------|--------------|-------------------|--------|-------------------|
| 0 | Full page reloads on navigation | None | Traditional MPA | 500KB per navigation, 200-500ms delay |
| 1 | Page reloads break UX | React Router with `<BrowserRouter>` and `<Link>` | Instant navigation | 0 bytes per navigation, <16ms |
| 2 | Duplicated navigation code | Shared `<Layout>` with `<Outlet>` | DRY navigation | No additional cost |
| 3 | Static routes only | URL parameters with `useParams()` | Dynamic routes | No additional cost |
| 4 | No search/filtering | Query strings with `useSearchParams()` | Shareable filtered views | No additional cost |
| 5 | Flat navigation structure | Nested routes with section layouts | Hierarchical organization | Minimal (layout components) |
| 6 | No access control | Protected routes with auth context | Secure user areas | Minimal (auth context) |
| 7 | No role-based access | Role checking in `<ProtectedRoute>` | Granular permissions | No additional cost |

### Final Implementation

Here's our complete, production-ready routing system:

**Project Structure**:
```
src/
├── components/
│   ├── Layout.tsx              ← Main site layout with auth-aware nav
│   ├── GuidesLayout.tsx        ← Guides section layout
│   ├── ApiLayout.tsx           ← API section layout
│   └── ProtectedRoute.tsx      ← Route protection with role checking
├── pages/
│   ├── Home.tsx                ← Public home page
│   ├── Login.tsx               ← Authentication page
│   ├── Dashboard.tsx           ← Protected user dashboard
│   ├── Settings.tsx            ← Protected user settings
│   ├── AdminPanel.tsx          ← Admin-only page
│   ├── Docs.tsx                ← Documentation index
│   ├── guides/                 ← Guides section pages
│   │   ├── GettingStarted.tsx
│   │   ├── Installation.tsx
│   │   └── Configuration.tsx
│   ├── api/                    ← API section pages
│   │   ├── Overview.tsx
│   │   ├── Authentication.tsx
│   │   └── Endpoints.tsx
│   ├── About.tsx               ← Public about page
│   └── NotFound.tsx            ← 404 page
├── context/
│   └── AuthContext.tsx         ← Authentication state management
├── data/
│   └── docs.ts                 ← Documentation content
├── App.tsx                     ← Route configuration
└── main.tsx                    ← App entry point
```

```tsx
// Complete App.tsx - Final version
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';
import { ProtectedRoute } from './components/ProtectedRoute';
import { Layout } from './components/Layout';
import { GuidesLayout } from './components/GuidesLayout';
import { ApiLayout } from './components/ApiLayout';
import { Home } from './pages/Home';
import { Login } from './pages/Login';
import { Dashboard } from './pages/Dashboard';
import { Settings } from './pages/Settings';
import { AdminPanel } from './pages/AdminPanel';
import { Docs } from './pages/Docs';
import { GettingStarted } from './pages/guides/GettingStarted';
import { Installation } from './pages/guides/Installation';
import { Configuration } from './pages/guides/Configuration';
import { Overview } from './pages/api/Overview';
import { Authentication } from './pages/api/Authentication';
import { Endpoints } from './pages/api/Endpoints';
import { About } from './pages/About';
import { NotFound } from './pages/NotFound';

export function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          {/* Public login page (no layout) */}
          <Route path="/login" element={<Login />} />
          
          {/* All other routes use main layout */}
          <Route path="/" element={<Layout />}>
            {/* Public routes */}
            <Route index element={<Home />} />
            <Route path="docs" element={<Docs />} />
            <Route path="about" element={<About />} />
            
            {/* Nested guides section */}
            <Route path="guides" element={<GuidesLayout />}>
              <Route index element={<GettingStarted />} />
              <Route path="getting-started" element={<GettingStarted />} />
              <Route path="installation" element={<Installation />} />
              <Route path="configuration" element={<Configuration />} />
            </Route>
            
            {/* Nested API section */}
            <Route path="api" element={<ApiLayout />}>
              <Route index element={<Overview />} />
              <Route path="overview" element={<Overview />} />
              <Route path="authentication" element={<Authentication />} />
              <Route path="endpoints" element={<Endpoints />} />
            </Route>
            
            {/* Protected routes - any authenticated user */}
            <Route 
              path="dashboard" 
              element={
                <ProtectedRoute>
                  <Dashboard />
                </ProtectedRoute>
              } 
            />
            <Route 
              path="settings" 
              element={
                <ProtectedRoute>
                  <Settings />
                </ProtectedRoute>
              } 
            />
            
            {/* Admin-only route */}
            <Route 
              path="admin" 
              element={
                <ProtectedRoute requireRole="admin">
                  <AdminPanel />
                </ProtectedRoute>
              } 
            />
            
            {/* 404 catch-all */}
            <Route path="*" element={<NotFound />} />
          </Route>
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}
```

### Decision Framework: Which Routing Approach When?

#### Client-Side Routing (React Router)

**Use when**:
- Building a web application (not a content site)
- User will navigate frequently between pages
- Need to preserve state across navigation
- Want instant, app-like navigation
- Already using React

**Avoid when**:
- Building a static content site (use Next.js or Astro)
- SEO is critical and you can't do SSR
- Target browsers don't support History API

#### URL Parameters vs. Query Strings

**URL Parameters (`:id`)**:
- Required, structural data
- Defines the resource being viewed
- Clean, SEO-friendly URLs
- Example: `/docs/getting-started`, `/user/123`

**Query Strings (`?key=value`)**:
- Optional filtering, sorting, pagination
- Shareable search results
- Preserves UI state in URL
- Example: `/docs?search=hooks&category=basics`

#### Nested Routes

**Use when**:
- Building sections with their own navigation
- Need section-specific layouts
- Want to avoid duplicating layout code
- Have hierarchical content structure

**Avoid when**:
- Simple sites with flat navigation
- All pages share identical layout
- Section-specific navigation isn't needed

#### Protected Routes

**Use when**:
- Building apps with user accounts
- Need to restrict access to certain pages
- Different user roles with different permissions
- Want to preserve intended destination after login

**Avoid when**:
- Completely public applications
- Backend handles all authorization
- Simple apps where all pages are accessible

### Common Failure Modes and Their Signatures

#### Symptom: Blank page on navigation

**Browser behavior**:
- URL changes but content doesn't appear
- No error in console

**Console pattern**:
```
No routes matched location "/some-path"
```

**Root cause**: No route defined for that path
**Solution**: Add catch-all route with `path="*"`

#### Symptom: Full page reload on link click

**Browser behavior**:
- White flash on navigation
- Console clears
- Network tab shows full bundle reload

**Root cause**: Using `<a>` instead of `<Link>`
**Solution**: Replace all `<a href>` with `<Link to>`

#### Symptom: Protected route accessible without login

**Browser behavior**:
- Can access protected pages directly
- No redirect to login

**Root cause**: Forgot to wrap route in `<ProtectedRoute>`
**Solution**: Wrap protected routes in protection component

#### Symptom: Layout re-renders on every navigation

**Browser behavior**:
- Header/footer flash on navigation
- React DevTools shows layout unmounting

**Root cause**: Layout not properly nested in route hierarchy
**Solution**: Ensure layout is parent route with `<Outlet />`

#### Symptom: URL parameters undefined

**Browser behavior**:
- `useParams()` returns empty object
- Component can't access URL data

**Console pattern**:
```
TypeError: Cannot read property 'id' of undefined
```

**Root cause**: Route path doesn't include parameter placeholder
**Solution**: Add `:paramName` to route path

### Lessons Learned

**1. Client-side routing transforms user experience**
- Eliminates page reloads
- Preserves JavaScript state
- Enables instant navigation
- Reduces bandwidth usage

**2. Route hierarchy should match URL structure**
- Nested routes create nested layouts
- Each level can have its own `<Outlet />`
- Layouts persist at their nesting level

**3. URL is state**
- Parameters for required data
- Query strings for optional filters
- Both are shareable and bookmarkable

**4. Protection is about UX, not security**
- Client-side checks prevent accidental access
- Always validate permissions on backend
- Use for user experience, not security

**5. Composition over configuration**
- Routes are components
- Layouts are components
- Protection is a wrapper component
- Everything composes naturally

### Next Steps

In the next chapter, we'll explore **Advanced Routing Patterns**:
- Code splitting and lazy loading routes
- Route-based data loading
- Scroll restoration and focus management
- Prefetching for instant navigation
- Optimizing bundle size with route-based splitting

But first, you have a solid foundation in React Router essentials. You can build complex, multi-page applications with nested layouts, dynamic routes, and protected areas—all with instant, app-like navigation.
