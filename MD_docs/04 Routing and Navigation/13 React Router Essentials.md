# Chapter 13: React Router Essentials

## Client-side routing basics

## The Problem: A "Single Page" Without Pages

Imagine a traditional website from the early 2000s. You click a link, the screen goes white, a spinner appears in your browser tab, and a completely new HTML document is loaded from the server. This is called server-side routing. It's robust, but it can feel slow and disjointed, like flipping through a deck of cards rather than interacting with a fluid application.

Modern web applications, especially those built with React, aim for a smoother, app-like experience. This is achieved through a Single-Page Application (SPA) architecture. In a SPA, the server sends the initial HTML, CSS, and JavaScript bundle just once. From then on, JavaScript takes over, dynamically rewriting parts of the page as the user interacts with it, without requesting a full new page from the server.

This creates a new problem: if we're not loading new pages, how do we create the illusion of different "pages" or views? How do we make URLs like `/about` or `/products/123` work? How do users share links or use their browser's back and forward buttons?

### A Note on Next.js

This chapter focuses on `react-router-dom`, the de-facto standard for routing in client-side rendered (CSR) React applications, such as those created with Vite or Create React App. Next.js provides its own powerful, file-system-based router out of the box. While the core concepts of routing are similar (URLs mapping to components), the implementation is different. We will cover Next.js's App Router in detail in Chapter 15. Understanding React Router is still immensely valuable as it teaches the fundamental principles of client-side navigation and is widely used in the React ecosystem outside of Next.js.

## Iteration 0: The Naive Approach with State

Let's build a reference application: a simple product catalog. We'll have a Home page, a Products list page, and an About page.

A beginner's first instinct might be to manage the "current page" using React state. Let's build this naive version to see exactly why it fails.

First, let's set up our project structure and components.

**Project Structure**:
```
src/
├── components/
│   ├── Home.tsx
│   ├── Products.tsx
│   └── About.tsx
└── App.tsx
```

**Component Code**:
We'll create three simple components to represent our pages.

```tsx
// src/components/Home.tsx
export default function Home() {
  return <h1>Welcome to Our Store!</h1>;
}
```

```tsx
// src/components/Products.tsx
export default function Products() {
  return <h1>Our Products</h1>;
}
```

```tsx
// src/components/About.tsx
export default function About() {
  return <h1>About Us</h1>;
}
```

Now, let's wire them together in `App.tsx` using state to control which component is visible.

```tsx
// src/App.tsx - Iteration 0: State-based "Routing"
import { useState } from 'react';
import Home from './components/Home';
import Products from './components/Products';
import About from './components/About';

type Page = 'home' | 'products' | 'about';

function App() {
  const [currentPage, setCurrentPage] = useState<Page>('home');

  const renderPage = () => {
    switch (currentPage) {
      case 'home':
        return <Home />;
      case 'products':
        return <Products />;
      case 'about':
        return <About />;
      default:
        return <Home />;
    }
  };

  return (
    <div>
      <nav>
        <button onClick={() => setCurrentPage('home')}>Home</button>
        <button onClick={() => setCurrentPage('products')}>Products</button>
        <button onClick={() => setCurrentPage('about')}>About</button>
      </nav>
      <hr />
      <main>
        {renderPage()}
      </main>
    </div>
  );
}

export default App;
```

This code *works* in the sense that you can click the buttons and the content changes. But it fails as a web application. Let's diagnose why.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
- The UI updates correctly when you click the buttons.
- The URL in the browser's address bar **never changes**. It always stays at `http://localhost:3000/`.
- Clicking the browser's "Back" button doesn't navigate to the previous view; it takes you to the site you were on before you loaded the React app.
- Refreshing the page (e.g., while on the "About" view) resets the application to the "Home" view, because the `currentPage` state is re-initialized to `'home'`.
- You cannot share a link to a specific page. If you copy the URL and send it to someone, they will always land on the Home page.

**Browser Console Output**:
```
(No errors or warnings)
```
The console is silent because, from React's perspective, nothing is wrong. We are just updating state and re-rendering, which is what React is designed to do.

**React DevTools Evidence**:
- **Component Tree**: The `App` component's `currentPage` state hook updates correctly on button clicks (e.g., from `'home'` to `'products'`).
- **Render Count**: Only the `App` component and the currently rendered page component re-render on navigation. This part is efficient.

**Let's parse this evidence**:

1.  **What the user experiences**: The application feels broken. Core browser features like the back button, refresh, and link sharing do not work as expected.

    -   Expected: The URL should update to reflect the current view (e.g., `/products`), and browser navigation should work.
    -   Actual: The URL is static, and browser navigation is broken.

2.  **What the console reveals**: Nothing. This is a logical failure, not a syntax or runtime error.

3.  **What DevTools shows**: React is doing its job correctly based on the state we've given it. The problem isn't in React's rendering logic.

4.  **Root cause identified**: The application's internal state (`currentPage`) is completely disconnected from the browser's URL and its history stack. The UI is a black box to the browser.

5.  **Why the current approach can't solve this**: We have no mechanism to read from or write to the browser's address bar. We are not participating in the browser's navigation system; we are fighting against it.

6.  **What we need**: A way to synchronize our component state with the browser's URL. We need a library that listens for URL changes (from user typing, back/forward buttons) and maps them to specific components, and also updates the URL when we navigate within the app. This is precisely what a client-side router does.

## Routes, links, and navigation

## Iteration 1: Introducing React Router

React Router is the standard library for routing in React. It provides a collection of components and hooks that connect your application's UI to the browser's URL.

### Setup

First, let's add it to our project.

```bash
npm install react-router-dom
```

Now, we'll refactor our `App.tsx` to use React Router's core components: `BrowserRouter`, `Routes`, and `Route`.

-   `BrowserRouter`: This component should wrap your entire application. It uses the HTML5 History API (pushState, replaceState, and the popstate event) to keep your UI in sync with the URL.
-   `Routes`: This component is a container for a collection of `Route`s. It intelligently looks at all its child `Route` elements to find the best match for the current URL.
-   `Route`: This is the core mapping component. It takes a `path` prop (the URL path) and an `element` prop (the React component to render when the path matches).

### Refactoring `App.tsx`

Let's replace our state-based logic with a proper routing setup.

**Before** (Iteration 0):

```tsx
// src/App.tsx - State-based "Routing"
import { useState } from 'react';
import Home from './components/Home';
import Products from './components/Products';
import About from './components/About';

type Page = 'home' | 'products' | 'about';

function App() {
  const [currentPage, setCurrentPage] = useState<Page>('home');
  // ... state-based rendering logic ...
}
```

**After** (Iteration 1):
We need to wrap our `App` in `BrowserRouter`. This is typically done in the entry point of your application, `src/index.tsx` or `src/main.tsx`.

```tsx
// src/index.tsx (or main.tsx)
import React from 'react';
import ReactDOM from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App';

const root = ReactDOM.createRoot(
  document.getElementById('root') as HTMLElement
);
root.render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>
);
```

Now, we can define our routes in `App.tsx`.

```tsx
// src/App.tsx - Iteration 1: Basic Routing
import { Routes, Route, Link } from 'react-router-dom';
import Home from './components/Home';
import Products from './components/Products';
import About from './components/About';

function App() {
  return (
    <div>
      <nav>
        {/* Replace buttons with Link components */}
        <Link to="/">Home</Link> |{' '}
        <Link to="/products">Products</Link> |{' '}
        <Link to="/about">About</Link>
      </nav>
      <hr />
      <main>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/products" element={<Products />} />
          <Route path="/about" element={<About />} />
        </Routes>
      </main>
    </div>
  );
}

export default App;
```

### The `<Link>` Component

Notice we replaced the `<button>` elements with `<Link>`. This is critical.

-   An `<a>` tag (`<a href="/products">`) would trigger a full page reload, defeating the purpose of a SPA.
-   A `<button>` with an `onClick` handler is what we used in our naive approach, which doesn't update the URL.
-   The `<Link>` component from `react-router-dom` is the solution. When clicked, it:
    1.  Prevents the browser's default behavior of a full page navigation.
    2.  Updates the URL in the address bar using the History API.
    3.  Notifies the rest of React Router that the URL has changed, causing the `<Routes>` component to re-evaluate and render the matching `<Route>`.

**Verification**:
Run the app now.
-   Clicking the "Products" link changes the URL to `/products` without a page refresh.
-   The `<Products />` component is rendered.
-   The browser's back and forward buttons now work correctly, cycling through your navigation history.
-   You can refresh the page at `/products`, and it will correctly load with the `<Products />` component visible.
-   You can copy the URL `/products`, open it in a new tab, and it will take you directly to the products page.

We have successfully solved all the problems from our initial implementation.

### Programmatic Navigation with `useNavigate`

Sometimes you need to navigate based on an action, not just a user clicking a link. For example, after a user submits a form, you might want to redirect them to their dashboard. This is called programmatic navigation.

React Router provides the `useNavigate` hook for this.

Let's add a "Go Home" button to our `About` page to demonstrate.

```tsx
// src/components/About.tsx
import { useNavigate } from 'react-router-dom';

export default function About() {
  const navigate = useNavigate();

  const handleGoHomeClick = () => {
    console.log('Navigating home...');
    navigate('/'); // Navigate to the home page
  };

  return (
    <div>
      <h1>About Us</h1>
      <button onClick={handleGoHomeClick}>Go Home</button>
    </div>
  );
}
```

When you click the "Go Home" button, the `navigate` function is called, which updates the URL and triggers the route change, just as if you had clicked a `<Link to="/">`.

## URL parameters and query strings

## Iteration 2: Handling Dynamic Data

Our product catalog is static. We have a single page for all products. In a real application, we'd want to click on a product and see a detailed view for that specific item. We can't create a new route in our code for every single product in our database. We need dynamic routes.

This is where URL parameters come in. A URL parameter is a dynamic segment of a URL path. For example, in `/products/123`, `123` is a parameter representing the product's ID.

### Setting up Product Data

First, let's create some mock product data.

```typescript
// src/data/products.ts
export interface Product {
  id: number;
  name: string;
  description: string;
  price: number;
}

export const products: Product[] = [
  { id: 1, name: 'React T-Shirt', description: 'A cool shirt for React devs.', price: 25.99 },
  { id: 2, name: 'Vue Mug', description: 'A mug for the progressive framework.', price: 15.50 },
  { id: 3, name: 'Angular Cap', description: 'A cap for enterprise-grade apps.', price: 20.00 },
];

export const getProductById = (id: number): Product | undefined => {
  return products.find(p => p.id === id);
};
```

Now, let's update our `Products` component to list these products and link to their detail pages.

```tsx
// src/components/Products.tsx
import { Link } from 'react-router-dom';
import { products } from '../data/products';

export default function Products() {
  return (
    <div>
      <h1>Our Products</h1>
      <ul>
        {products.map(product => (
          <li key={product.id}>
            <Link to={`/products/${product.id}`}>
              {product.name}
            </Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

If you run this now and click on a product link, you'll get a blank page. Why?

### Diagnostic Analysis: The Unmatched Route

**Browser Behavior**:
- You are on the `/products` page and see the list of products.
- You click on "React T-Shirt".
- The URL changes to `/products/1`.
- The main content area of the page becomes blank. The navigation links are still visible.

**Browser Console Output**:
```
Warning: No routes matched location "/products/1"
```
React Router gives us a very helpful warning here.

**Let's parse this evidence**:

1.  **What the user experiences**: A broken link. The app navigates to a new URL, but nothing renders.

2.  **What the console reveals**: The root cause is explicitly stated: React Router has no `Route` definition that matches the path `/products/1`. Our current routes are `/`, `/products`, and `/about`.

3.  **Root cause identified**: We have not defined a route that can handle dynamic segments like `/products/:id`.

4.  **What we need**: A way to define a route pattern that matches any product ID, and a way to extract that ID from the URL within the component so we can fetch the correct data.

### Implementing Dynamic Routes with `useParams`

React Router allows you to define URL parameters in your route path using a colon, like `:productId`.

First, let's create the `ProductDetail` component. It will use the `useParams` hook to access the dynamic parts of the URL.

```tsx
// src/components/ProductDetail.tsx
import { useParams } from 'react-router-dom';
import { getProductById } from '../data/products';

export default function ProductDetail() {
  // useParams returns an object of key/value pairs of URL parameters
  // e.g., for a path "/products/1", params will be { productId: "1" }
  const { productId } = useParams<{ productId: string }>();

  // Note: URL params are always strings! We need to parse it.
  const product = getProductById(Number(productId));

  if (!product) {
    return <h2>Product not found!</h2>;
  }

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p>Price: ${product.price.toFixed(2)}</p>
    </div>
  );
}
```

Now, we add the new dynamic route to our `App.tsx`.

```tsx
// src/App.tsx - Iteration 2: Adding a dynamic route
import { Routes, Route, Link } from 'react-router-dom';
import Home from './components/Home';
import Products from './components/Products';
import About from './components/About';
import ProductDetail from './components/ProductDetail'; // <-- Import

function App() {
  return (
    <div>
      <nav>
        {/* ... nav links ... */}
      </nav>
      <hr />
      <main>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/products" element={<Products />} />
          {/* This is the dynamic route. :productId is a URL parameter. */}
          <Route path="/products/:productId" element={<ProductDetail />} />
          <Route path="/about" element={<About />} />
        </Routes>
      </main>
    </div>
  );
}

export default App;
```

**Verification**:
Now, when you click a product link:
1.  The URL changes to `/products/1`.
2.  React Router matches this URL with the `<Route path="/products/:productId" ... />`.
3.  It renders the `<ProductDetail />` component.
4.  Inside `ProductDetail`, `useParams()` returns `{ productId: '1' }`.
5.  The component uses this ID to fetch and display the correct product data.

### Filtering with Query Strings

What if we want to filter or sort our products? For example, `/products?sort=price&order=desc`. These key-value pairs after the `?` are called query strings (or search parameters). They are ideal for optional data that modifies the state of a view.

React Router provides the `useSearchParams` hook to work with them. It works much like `useState`, returning the current search params and a function to update them.

Let's modify our `Products` component to allow sorting.

```tsx
// src/components/Products.tsx
import { Link, useSearchParams } from 'react-router-dom';
import { products, Product } from '../data/products';

export default function Products() {
  const [searchParams, setSearchParams] = useSearchParams();
  const sortOrder = searchParams.get('sort'); // e.g., 'asc' or 'desc'

  const sortedProducts = [...products].sort((a, b) => {
    if (sortOrder === 'asc') {
      return a.name.localeCompare(b.name);
    }
    if (sortOrder === 'desc') {
      return b.name.localeCompare(a.name);
    }
    return 0; // No sorting
  });

  const handleSort = (order: 'asc' | 'desc') => {
    setSearchParams({ sort: order });
  };

  return (
    <div>
      <h1>Our Products</h1>
      <div>
        <button onClick={() => handleSort('asc')}>Sort A-Z</button>
        <button onClick={() => handleSort('desc')}>Sort Z-A</button>
      </div>
      <ul>
        {sortedProducts.map(product => (
          <li key={product.id}>
            <Link to={`/products/${product.id}`}>
              {product.name}
            </Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

Now, when you click the sort buttons:
-   `setSearchParams` is called, which updates the URL to `/products?sort=asc` (for example).
-   This causes the `Products` component to re-render.
-   On re-render, `useSearchParams` provides the new `searchParams` object.
-   `searchParams.get('sort')` reads the value, and the sorting logic is applied.

This is powerful because the filtered/sorted state is now encoded in the URL, making it shareable and bookmarkable.

## Nested routes and layouts

## Iteration 3: Creating a Consistent Layout

Look at our `App.tsx`. The `nav` and `hr` elements are present on every single page. What if we wanted a more complex layout, like a header, sidebar, and footer? We would have to include those components inside `Home`, `Products`, `About`, etc. This violates the DRY (Don't Repeat Yourself) principle.

### The Problem: Repetitive Layouts

Currently, our layout is defined once in `App.tsx`. But if the layout itself were a component, we'd be tempted to do this:

```tsx
// A common but inefficient pattern
function Home() {
  return (
    <Layout>
      <h1>Welcome</h1>
    </Layout>
  );
}

function Products() {
  return (
    <Layout>
      <h1>Our Products</h1>
    </Layout>
  );
}
```

This is bad because the `Layout` component would unmount and remount on every navigation, potentially losing its own state (like a search term in a header search bar).

React Router provides a much more elegant solution: **nested routes** and the `<Outlet />` component.

The idea is to create a parent "layout route" that renders the shared UI, and then specify a placeholder where child routes should be rendered.

### Implementing a Layout Route

First, let's create a `Layout` component. The special part is the `<Outlet />` component from `react-router-dom`. It acts as a placeholder, rendering the matched child route's element.

```tsx
// src/components/Layout.tsx
import { Link, Outlet } from 'react-router-dom';

export default function Layout() {
  return (
    <div>
      <header>
        <h1>My Awesome Store</h1>
        <nav>
          <Link to="/">Home</Link> |{' '}
          <Link to="/products">Products</Link> |{' '}
          <Link to="/about">About</Link>
        </nav>
      </header>
      <hr />
      <main>
        {/* Child routes will be rendered here */}
        <Outlet />
      </main>
      <hr />
      <footer>
        <p>© 2024 My Awesome Store</p>
      </footer>
    </div>
  );
}
```

Now, we refactor our `App.tsx` to use this layout route. We'll nest the page routes *inside* the layout route.

```tsx
// src/App.tsx - Iteration 3: Nested Routes
import { Routes, Route } from 'react-router-dom';
import Home from './components/Home';
import Products from './components/Products';
import About from './components/About';
import ProductDetail from './components/ProductDetail';
import Layout from './components/Layout'; // <-- Import

function App() {
  return (
    <Routes>
      {/* The Layout component is now the element for the parent route */}
      <Route path="/" element={<Layout />}>
        {/* Child routes. Their paths are relative to the parent. */}
        {/* The 'index' route renders at the parent's path ('/') */}
        <Route index element={<Home />} />
        <Route path="products" element={<Products />} />
        <Route path="products/:productId" element={<ProductDetail />} />
        <Route path="about" element={<About />} />
      </Route>
    </Routes>
  );
}

export default App;
```

### Key Concepts in this Refactor

1.  **Parent Route**: We create a `<Route path="/" element={<Layout />}>`. This route will always match when the URL starts with `/`.
2.  **Child Routes**: The other routes are nested inside. Their paths are now relative to the parent. For example, `path="products"` combined with the parent's `/` becomes `/products`.
3.  **Index Route**: The `<Route index ... />` is special. It renders when the URL exactly matches the parent's path. So, when the user is at `/`, the `<Layout />` renders, and its `<Outlet />` is filled with the `<Home />` component.
4.  **`<Outlet />`**: This is the magic. The `<Layout />` component renders the header, nav, and footer. The `<Outlet />` component within it is replaced by `<Home />`, `<Products />`, or `<About />` depending on which child route matches.

**Verification**:
The application looks and behaves exactly as before, but our code is now much cleaner and more maintainable. The `Layout` component will persist across navigations between its child routes, preserving any state it might have.

### Common Failure Modes and Their Signatures

#### Symptom: Layout renders, but the page content is blank.

**Browser behavior**:
You see the header and footer from your `Layout` component, but the main content area where the page component should be is empty.

**Console pattern**:
```
(No errors or warnings)
```

**DevTools clues**:
- In the Components tab, you see your `Layout` component in the tree, but the component you expect to see inside it (e.g., `Home`) is not there.

**Root cause**: You forgot to add `<Outlet />` inside your layout component. Without it, React Router has nowhere to render the child route's element.

**Solution**: Ensure your `Layout.tsx` component includes `<Outlet />` where you want the child content to appear.

## Protected routes

## Iteration 4: Protecting Content

Most applications have areas that should only be accessible to authenticated users, like a user profile or an admin dashboard. We need a way to create "protected routes".

### The Goal

We want to add an `/admin` route. If a user is logged in, they should see the admin dashboard. If they are not logged in and try to access `/admin`, they should be redirected to a `/login` page.

### Simulating Authentication

For this example, we won't build a full authentication system. We'll use React Context to provide a simple, app-wide "auth" state.

```tsx
// src/context/AuthContext.tsx
import { createContext, useContext, useState, ReactNode } from 'react';

interface AuthContextType {
  isAuthenticated: boolean;
  login: () => void;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [isAuthenticated, setIsAuthenticated] = useState(false);

  const login = () => setIsAuthenticated(true);
  const logout = () => setIsAuthenticated(false);

  return (
    <AuthContext.Provider value={{ isAuthenticated, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

We need to wrap our application with this provider. The best place is in `index.tsx`, surrounding the `BrowserRouter`.

```tsx
// src/index.tsx
// ... imports
import { AuthProvider } from './context/AuthContext';

root.render(
  <React.StrictMode>
    <AuthProvider>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </AuthProvider>
  </React.StrictMode>
);
```

Next, let's create the `Login` and `AdminDashboard` components.

```tsx
// src/components/Login.tsx
import { useNavigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export default function Login() {
  const navigate = useNavigate();
  const { login } = useAuth();

  const handleLogin = () => {
    login();
    navigate('/admin'); // Redirect to admin after login
  };

  return (
    <div>
      <h2>Login Page</h2>
      <p>You must log in to view the admin page.</p>
      <button onClick={handleLogin}>Log In</button>
    </div>
  );
}
```

```tsx
// src/components/AdminDashboard.tsx
import { useAuth } from '../context/AuthContext';
import { useNavigate } from 'react-router-dom';

export default function AdminDashboard() {
  const { logout } = useAuth();
  const navigate = useNavigate();

  const handleLogout = () => {
    logout();
    navigate('/');
  };

  return (
    <div>
      <h2>Admin Dashboard</h2>
      <p>Welcome, admin!</p>
      <button onClick={handleLogout}>Log Out</button>
    </div>
  );
}
```

### Creating the `ProtectedRoute` Component

Now for the core logic. We'll create a wrapper component called `ProtectedRoute`. Its job is to check the authentication status. If the user is authenticated, it renders its `children`. If not, it uses the `<Navigate>` component from React Router to perform a declarative redirect.

```tsx
// src/components/ProtectedRoute.tsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export default function ProtectedRoute({ children }: { children: JSX.Element }) {
  const { isAuthenticated } = useAuth();

  if (!isAuthenticated) {
    // Redirect them to the /login page, but save the current location they were
    // trying to go to. This allows us to send them back after they log in.
    return <Navigate to="/login" replace />;
  }

  return children;
}
```

The `<Navigate>` component is special. When rendered, it immediately changes the location. The `replace` prop is important; it replaces the current entry in the history stack instead of pushing a new one. This means when the user clicks the "back" button from the login page, they won't go back to the protected page they were just redirected from (which would cause another redirect loop).

### Updating the Routes

Finally, let's add the new routes to `App.tsx` and wrap the admin route with our `ProtectedRoute`.

```tsx
// src/App.tsx - Iteration 4: Protected Routes
import { Routes, Route } from 'react-router-dom';
// ... other component imports
import Login from './components/Login';
import AdminDashboard from './components/AdminDashboard';
import ProtectedRoute from './components/ProtectedRoute';

function App() {
  return (
    <Routes>
      <Route path="/" element={<Layout />}>
        <Route index element={<Home />} />
        <Route path="products" element={<Products />} />
        <Route path="products/:productId" element={<ProductDetail />} />
        <Route path="about" element={<About />} />
        
        {/* Public login route */}
        <Route path="login" element={<Login />} />

        {/* Protected admin route */}
        <Route 
          path="admin" 
          element={
            <ProtectedRoute>
              <AdminDashboard />
            </ProtectedRoute>
          } 
        />
      </Route>
    </Routes>
  );
}

export default App;
```

**Verification**:
1.  Try to navigate directly to `/admin` by typing it in the URL bar. You will be instantly redirected to `/login`.
2.  Click the "Log In" button on the `/login` page.
3.  You are now redirected to `/admin` and can see the dashboard.
4.  Refresh the page. You stay on the admin dashboard (because our simple auth state is lost on refresh, a real app would use localStorage or cookies to persist it, but the routing logic holds).
5.  Click "Log Out". You are redirected back to the home page. If you try to go to `/admin` again, you are sent back to `/login`.

The protection mechanism works perfectly.

### The Journey: From Problem to Solution

| Iteration | Failure Mode                                      | Technique Applied                               | Result                                                              |
| --------- | ------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------------- |
| 0         | No shareable URLs, no browser history integration | State-based component switching                 | A "working" UI that breaks fundamental web expectations.            |
| 1         | Full page reloads with `<a>` tags                 | `BrowserRouter`, `Routes`, `Route`, `Link`      | True client-side navigation with URL synchronization.               |
| 2         | Cannot represent unique items (e.g., products)    | URL Parameters (`:id`) and `useParams` hook     | Dynamic pages based on URL segments.                                |
| 2.5       | Cannot represent view state (e.g., sorting)       | Query Strings (`?sort=`) and `useSearchParams`  | Shareable URLs that capture the state of the UI (filters, sorts).   |
| 3         | Repetitive layout code in every page component    | Nested Routes and the `<Outlet />` component    | Clean, maintainable layouts that persist state across navigations.  |
| 4         | All pages are publicly accessible                 | Custom `ProtectedRoute` with `<Navigate>`       | Secure sections of the app accessible only to authenticated users.  |

### Final Implementation

Here is the final, complete `App.tsx` incorporating all the concepts we've learned.

```tsx
// src/App.tsx - Final Version
import { Routes, Route } from 'react-router-dom';

// Layout and Page Components
import Layout from './components/Layout';
import Home from './components/Home';
import Products from './components/Products';
import ProductDetail from './components/ProductDetail';
import About from './components/About';
import Login from './components/Login';
import AdminDashboard from './components/AdminDashboard';

// Auth Wrapper
import ProtectedRoute from './components/ProtectedRoute';

function App() {
  return (
    <Routes>
      {/* Parent layout route */}
      <Route path="/" element={<Layout />}>
        {/* Public child routes */}
        <Route index element={<Home />} />
        <Route path="products" element={<Products />} />
        <Route path="products/:productId" element={<ProductDetail />} />
        <Route path="about" element={<About />} />
        <Route path="login" element={<Login />} />

        {/* Protected child route */}
        <Route 
          path="admin" 
          element={
            <ProtectedRoute>
              <AdminDashboard />
            </ProtectedRoute>
          } 
        />
        
        {/* Optional: Add a 404 Not Found route */}
        <Route path="*" element={<h2>404: Page Not Found</h2>} />
      </Route>
    </Routes>
  );
}

export default App;
```

This structure provides a robust, maintainable, and feature-complete routing setup for a client-side React application, serving as a strong foundation for building complex SPAs.
