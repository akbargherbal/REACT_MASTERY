# Chapter 22: Error Boundaries and Error Handling

## Error boundaries in React

## The Failure: When One Component Crashes the Entire App

Let's start with a realistic scenario that will serve as our reference implementation throughout this chapter. We're building a product dashboard that displays multiple widgets: user statistics, recent orders, and a revenue chart. Each widget fetches its own data independently.

Here's our initial implementation:

```tsx
// src/app/dashboard/page.tsx
'use client';

import { useState, useEffect } from 'react';

interface UserStats {
  totalUsers: number;
  activeUsers: number;
  newToday: number;
}

function UserStatsWidget() {
  const [stats, setStats] = useState<UserStats | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/stats/users')
      .then(res => res.json())
      .then(data => {
        setStats(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <div className="widget">Loading user stats...</div>;

  // This will crash if stats.totalUsers is undefined
  return (
    <div className="widget">
      <h2>User Statistics</h2>
      <p>Total Users: {stats.totalUsers.toLocaleString()}</p>
      <p>Active: {stats.activeUsers.toLocaleString()}</p>
      <p>New Today: {stats.newToday.toLocaleString()}</p>
    </div>
  );
}

function RecentOrdersWidget() {
  const [orders, setOrders] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/orders/recent')
      .then(res => res.json())
      .then(data => {
        setOrders(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <div className="widget">Loading orders...</div>;

  return (
    <div className="widget">
      <h2>Recent Orders</h2>
      <ul>
        {orders.map((order: any) => (
          <li key={order.id}>
            Order #{order.id} - ${order.total}
          </li>
        ))}
      </ul>
    </div>
  );
}

function RevenueChartWidget() {
  const [revenue, setRevenue] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/stats/revenue')
      .then(res => res.json())
      .then(data => {
        setRevenue(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <div className="widget">Loading revenue...</div>;

  return (
    <div className="widget">
      <h2>Revenue Chart</h2>
      <p>Today: ${revenue.today}</p>
      <p>This Week: ${revenue.week}</p>
      <p>This Month: ${revenue.month}</p>
    </div>
  );
}

export default function DashboardPage() {
  return (
    <div className="dashboard">
      <h1>Dashboard</h1>
      <div className="widgets-grid">
        <UserStatsWidget />
        <RecentOrdersWidget />
        <RevenueChartWidget />
      </div>
    </div>
  );
}
```

Now let's see what happens when the user stats API returns malformed data. We'll simulate this by having the API return `null` for `totalUsers`:

```json
// API response from /api/stats/users (malformed)
{
  "totalUsers": null,
  "activeUsers": 1250,
  "newToday": 42
}
```

### Diagnostic Analysis: Reading the Catastrophic Failure

**Browser Behavior**:
The entire dashboard disappears. The user sees a blank white screen with no indication of what went wrong. All three widgets vanish, even though only one had a problem.

**Browser Console Output**:
```
Uncaught TypeError: Cannot read properties of null (reading 'toLocaleString')
    at UserStatsWidget (page.tsx:23)
    at renderWithHooks (react-dom.development.js:16305)
    at updateFunctionComponent (react-dom.development.js:19588)
    at beginWork (react-dom.development.js:21601)

The above error occurred in the <UserStatsWidget> component:
    at UserStatsWidget (http://localhost:3000/dashboard/page.tsx:12:5)
    at div
    at DashboardPage (http://localhost:3000/dashboard/page.tsx:65:3)

Consider adding an error boundary to your tree to customize error handling behavior.
```

**React DevTools Evidence**:
- Component tree shows `DashboardPage` rendered
- `UserStatsWidget`, `RecentOrdersWidget`, and `RevenueChartWidget` all show as unmounted
- No components visible in the tree after the error
- Props/State inspection unavailable (components unmounted)

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: See the dashboard with three widgets, possibly with one showing an error
   - Actual: Blank screen, no content, no error message, no way to recover

2. **What the console reveals**:
   - Key indicator: `Cannot read properties of null (reading 'toLocaleString')`
   - Error location: `UserStatsWidget` at line 23
   - The error occurred when trying to call `.toLocaleString()` on `null`

3. **What DevTools shows**:
   - Component state: All components unmounted after error
   - Render behavior: React stopped rendering the entire tree
   - The error in one component caused React to unmount everything

4. **Root cause identified**: 
   When `stats.totalUsers` is `null`, calling `.toLocaleString()` throws an error. React's default behavior is to unmount the entire component tree when an unhandled error occurs during rendering.

5. **Why the current approach can't solve this**:
   We have no mechanism to catch rendering errors. Try-catch blocks don't work in React components because rendering is asynchronous. We need a React-specific error handling mechanism.

6. **What we need**:
   A way to catch errors in component rendering and prevent them from crashing the entire application. React provides **Error Boundaries** for exactly this purpose.

## Understanding Error Boundaries

Error boundaries are React components that catch JavaScript errors anywhere in their child component tree, log those errors, and display a fallback UI instead of crashing the entire application.

**Key characteristics**:
- Error boundaries catch errors during **rendering**, in **lifecycle methods**, and in **constructors** of child components
- They do NOT catch errors in event handlers, async code (setTimeout, promises), or Server Components
- They must be class components (as of React 18, no hooks equivalent exists)
- They work by implementing `static getDerivedStateFromError()` or `componentDidCatch()`

### Creating a Basic Error Boundary

Let's create our first error boundary:

```tsx
// src/components/ErrorBoundary.tsx
'use client';

import React, { Component, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = {
      hasError: false,
      error: null
    };
  }

  // This lifecycle method is called when an error is thrown
  static getDerivedStateFromError(error: Error): State {
    // Update state so the next render shows the fallback UI
    return {
      hasError: true,
      error
    };
  }

  // This lifecycle method is called after an error is caught
  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    // Log the error to console (we'll add proper logging later)
    console.error('Error caught by boundary:', error);
    console.error('Component stack:', errorInfo.componentStack);
  }

  render() {
    if (this.state.hasError) {
      // Render custom fallback UI
      if (this.props.fallback) {
        return this.props.fallback;
      }

      return (
        <div className="error-boundary-fallback">
          <h2>Something went wrong</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

Now let's wrap each widget in its own error boundary:

```tsx
// src/app/dashboard/page.tsx
'use client';

import { useState, useEffect } from 'react';
import { ErrorBoundary } from '@/components/ErrorBoundary';

// ... (UserStatsWidget, RecentOrdersWidget, RevenueChartWidget remain the same)

export default function DashboardPage() {
  return (
    <div className="dashboard">
      <h1>Dashboard</h1>
      <div className="widgets-grid">
        <ErrorBoundary fallback={<WidgetError name="User Stats" />}>
          <UserStatsWidget />
        </ErrorBoundary>
        
        <ErrorBoundary fallback={<WidgetError name="Recent Orders" />}>
          <RecentOrdersWidget />
        </ErrorBoundary>
        
        <ErrorBoundary fallback={<WidgetError name="Revenue Chart" />}>
          <RevenueChartWidget />
        </ErrorBoundary>
      </div>
    </div>
  );
}

function WidgetError({ name }: { name: string }) {
  return (
    <div className="widget widget-error">
      <h2>{name}</h2>
      <p className="error-message">Failed to load data</p>
      <button onClick={() => window.location.reload()}>
        Reload Dashboard
      </button>
    </div>
  );
}
```

### Verification: Error Boundary in Action

Now when the user stats API returns malformed data:

**Browser Behavior**:
- The dashboard remains visible
- User Stats widget shows the error fallback UI
- Recent Orders and Revenue Chart widgets continue to function normally
- User can still interact with working widgets

**Browser Console Output**:
```
Error caught by boundary: TypeError: Cannot read properties of null (reading 'toLocaleString')
Component stack:
    at UserStatsWidget (http://localhost:3000/dashboard/page.tsx:12:5)
    at ErrorBoundary (http://localhost:3000/components/ErrorBoundary.tsx:15:3)
    at div
    at DashboardPage (http://localhost:3000/dashboard/page.tsx:65:3)
```

**React DevTools Evidence**:
- `DashboardPage` component still mounted
- `ErrorBoundary` (wrapping UserStatsWidget) shows `hasError: true` in state
- `RecentOrdersWidget` and `RevenueChartWidget` continue rendering normally
- Component tree remains intact

**Expected vs. Actual improvement**:
- Before: Entire dashboard crashes, blank screen
- After: Only the failing widget shows an error, rest of the dashboard works
- User experience: Degraded but functional instead of completely broken

### Limitation Preview

This solves the catastrophic failure problem, but we still have issues:
1. The error message is generic and unhelpful
2. We're logging to console instead of a proper monitoring service
3. The "Try again" button doesn't actually retry the failed component
4. We have no way to differentiate between different types of errors
5. Error boundaries don't catch errors in event handlers or async code

Let's address these limitations in the next iteration.

## Iteration 2: Granular Error Boundaries with Recovery

**Current limitation**: Our error boundary is too generic. It treats all errors the same way and doesn't provide meaningful recovery options.

**New scenario**: What if we want different error handling strategies for different types of failures? Network errors should be retryable, but programming errors (like null reference) might need different handling.

Let's create a more sophisticated error boundary:

```tsx
// src/components/ErrorBoundary.tsx
'use client';

import React, { Component, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: (error: Error, reset: () => void) => ReactNode;
  onError?: (error: Error, errorInfo: React.ErrorInfo) => void;
  resetKeys?: Array<string | number>;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = {
      hasError: false,
      error: null
    };
  }

  static getDerivedStateFromError(error: Error): State {
    return {
      hasError: true,
      error
    };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    // Call custom error handler if provided
    if (this.props.onError) {
      this.props.onError(error, errorInfo);
    } else {
      console.error('Error caught by boundary:', error);
      console.error('Component stack:', errorInfo.componentStack);
    }
  }

  componentDidUpdate(prevProps: Props) {
    // Reset error state if resetKeys change
    if (this.state.hasError && this.props.resetKeys) {
      const prevKeys = prevProps.resetKeys || [];
      const currentKeys = this.props.resetKeys;
      
      const hasChanged = currentKeys.some((key, index) => key !== prevKeys[index]);
      
      if (hasChanged) {
        this.reset();
      }
    }
  }

  reset = () => {
    this.setState({
      hasError: false,
      error: null
    });
  };

  render() {
    if (this.state.hasError && this.state.error) {
      if (this.props.fallback) {
        return this.props.fallback(this.state.error, this.reset);
      }

      return (
        <div className="error-boundary-fallback">
          <h2>Something went wrong</h2>
          <p>{this.state.error.message}</p>
          <button onClick={this.reset}>Try again</button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

Now let's create widget-specific error handling with proper retry logic:

```tsx
// src/app/dashboard/page.tsx
'use client';

import { useState, useEffect } from 'react';
import { ErrorBoundary } from '@/components/ErrorBoundary';

interface UserStats {
  totalUsers: number;
  activeUsers: number;
  newToday: number;
}

function UserStatsWidget({ retryKey }: { retryKey: number }) {
  const [stats, setStats] = useState<UserStats | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    setLoading(true);
    fetch('/api/stats/users')
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}: ${res.statusText}`);
        return res.json();
      })
      .then(data => {
        // Validate data structure
        if (!data || typeof data.totalUsers !== 'number') {
          throw new Error('Invalid data format from API');
        }
        setStats(data);
        setLoading(false);
      })
      .catch(error => {
        setLoading(false);
        // Re-throw to be caught by error boundary
        throw error;
      });
  }, [retryKey]);

  if (loading) return <div className="widget">Loading user stats...</div>;

  return (
    <div className="widget">
      <h2>User Statistics</h2>
      <p>Total Users: {stats!.totalUsers.toLocaleString()}</p>
      <p>Active: {stats!.activeUsers.toLocaleString()}</p>
      <p>New Today: {stats!.newToday.toLocaleString()}</p>
    </div>
  );
}

// Similar updates for other widgets...

export default function DashboardPage() {
  const [userStatsRetry, setUserStatsRetry] = useState(0);
  const [ordersRetry, setOrdersRetry] = useState(0);
  const [revenueRetry, setRevenueRetry] = useState(0);

  return (
    <div className="dashboard">
      <h1>Dashboard</h1>
      <div className="widgets-grid">
        <ErrorBoundary
          resetKeys={[userStatsRetry]}
          fallback={(error, reset) => (
            <WidgetError
              name="User Stats"
              error={error}
              onRetry={() => {
                setUserStatsRetry(prev => prev + 1);
                reset();
              }}
            />
          )}
        >
          <UserStatsWidget retryKey={userStatsRetry} />
        </ErrorBoundary>
        
        <ErrorBoundary
          resetKeys={[ordersRetry]}
          fallback={(error, reset) => (
            <WidgetError
              name="Recent Orders"
              error={error}
              onRetry={() => {
                setOrdersRetry(prev => prev + 1);
                reset();
              }}
            />
          )}
        >
          <RecentOrdersWidget retryKey={ordersRetry} />
        </ErrorBoundary>
        
        <ErrorBoundary
          resetKeys={[revenueRetry]}
          fallback={(error, reset) => (
            <WidgetError
              name="Revenue Chart"
              error={error}
              onRetry={() => {
                setRevenueRetry(prev => prev + 1);
                reset();
              }}
            />
          )}
        >
          <RevenueChartWidget retryKey={revenueRetry} />
        </ErrorBoundary>
      </div>
    </div>
  );
}

function WidgetError({ 
  name, 
  error, 
  onRetry 
}: { 
  name: string; 
  error: Error; 
  onRetry: () => void;
}) {
  const isNetworkError = error.message.includes('HTTP') || 
                         error.message.includes('fetch');
  
  return (
    <div className="widget widget-error">
      <h2>{name}</h2>
      <p className="error-message">
        {isNetworkError 
          ? 'Failed to load data from server' 
          : 'An unexpected error occurred'}
      </p>
      <details className="error-details">
        <summary>Technical details</summary>
        {error.message}
```

### Verification: Improved Error Handling

Now when the user stats API fails:

**Browser Behavior**:
- User Stats widget shows a specific error message
- "Retry" button actually re-fetches the data
- Technical details are hidden but accessible
- Other widgets continue working

**Browser Console Output**:
```
Error caught by boundary: Error: Invalid data format from API
Component stack:
    at UserStatsWidget (http://localhost:3000/dashboard/page.tsx:15:5)
    at ErrorBoundary (http://localhost:3000/components/ErrorBoundary.tsx:18:3)
```

**React DevTools Evidence**:
- When retry button is clicked, `retryKey` prop changes from 0 to 1
- `UserStatsWidget` remounts with new `retryKey`
- `useEffect` runs again, fetching fresh data
- If successful, error boundary resets and widget displays normally

**Expected vs. Actual improvement**:
- Before: Generic error, no retry mechanism
- After: Specific error messages, functional retry, graceful degradation
- User experience: Clear feedback and recovery path

### Limitation Preview

We've improved error handling significantly, but there's still a critical gap: **we're only logging errors to the console**. In production, we need to:
1. Track errors in a centralized monitoring system
2. Get notified when errors occur
3. Analyze error patterns and frequency
4. Capture user context (browser, OS, user actions leading to error)

This is where error monitoring services like Sentry come in.

## Common Failure Modes and Their Signatures

### Symptom: Error boundary doesn't catch the error

**Browser behavior**:
Application still crashes with blank screen

**Console pattern**:
```
Uncaught Error: Something went wrong
    at handleClick (Component.tsx:45)
```

**DevTools clues**:
- Error occurs in event handler, not during render
- Component tree unmounts completely
- No error boundary state change visible

**Root cause**: Error boundaries only catch errors during rendering, in lifecycle methods, and in constructors. They don't catch:
- Errors in event handlers
- Async code (setTimeout, promises)
- Server-side rendering errors
- Errors in the error boundary itself

**Solution**: For event handlers and async code, use try-catch blocks:

```tsx
// Event handler errors - use try-catch
function MyComponent() {
  const handleClick = async () => {
    try {
      await riskyOperation();
    } catch (error) {
      // Handle error or set error state
      console.error('Operation failed:', error);
      setError(error);
    }
  };

  return <button onClick={handleClick}>Click me</button>;
}
```

### Symptom: Error boundary catches error but doesn't reset

**Browser behavior**:
Error fallback UI displays, but clicking "Try again" doesn't work

**Console pattern**:
```
Error caught by boundary: TypeError: ...
(No additional errors, but component doesn't remount)
```

**DevTools clues**:
- Error boundary state shows `hasError: true`
- Clicking reset button doesn't change state
- Child component never remounts

**Root cause**: The reset mechanism isn't properly implemented. Either:
1. The reset function isn't updating state correctly
2. The child component isn't keyed or doesn't respond to prop changes
3. The error condition still exists (e.g., bad data in parent state)

**Solution**: Use `resetKeys` prop to force remount:

```tsx
// Parent component manages retry state
function ParentComponent() {
  const [retryCount, setRetryCount] = useState(0);

  return (
    <ErrorBoundary
      resetKeys={[retryCount]}
      fallback={(error, reset) => (
        <div>
          <p>Error: {error.message}</p>
          <button onClick={() => {
            setRetryCount(prev => prev + 1);
            reset();
          }}>
            Try again
          </button>
        </div>
      )}
    >
      <ChildComponent key={retryCount} />
    </ErrorBoundary>
  );
}
```

### Symptom: Multiple error boundaries trigger for same error

**Browser behavior**:
Multiple error fallback UIs appear, nested inside each other

**Console pattern**:
```
Error caught by boundary: TypeError: ...
Error caught by boundary: TypeError: ...
Error caught by boundary: TypeError: ...
```

**DevTools clues**:
- Multiple error boundaries in component tree all show `hasError: true`
- Error propagates up through nested boundaries
- Innermost boundary should catch it, but doesn't

**Root cause**: Error boundaries don't stop error propagation by default. If an error boundary's fallback UI itself throws an error, it propagates to the next boundary up.

**Solution**: Ensure fallback UI is error-free and consider using a single boundary at appropriate level:

```tsx
// Bad: Nested boundaries that might all trigger
<ErrorBoundary>
  <ErrorBoundary>
    <ErrorBoundary>
      <RiskyComponent />
    </ErrorBoundary>
  </ErrorBoundary>
</ErrorBoundary>

// Good: Single boundary at appropriate level
<ErrorBoundary>
  <div className="widget-container">
    <RiskyComponent />
  </div>
</ErrorBoundary>

// Good: Multiple boundaries for independent sections
<div className="dashboard">
  <ErrorBoundary>
    <WidgetA />
  </ErrorBoundary>
  <ErrorBoundary>
    <WidgetB />
  </ErrorBoundary>
</div>
```

## Error handling in Next.js

## The Failure: Error Boundaries Don't Work in Server Components

**Current limitation**: Our error boundary works great for Client Components, but Next.js introduces a new complexity: Server Components. Let's see what happens when a Server Component fails.

**New scenario**: We're adding a server-rendered product list to our dashboard. The data is fetched on the server for better performance and SEO.

```tsx
// src/app/dashboard/products/page.tsx
// This is a Server Component (default in Next.js App Router)

interface Product {
  id: string;
  name: string;
  price: number;
  stock: number;
}

async function getProducts(): Promise<Product[]> {
  const res = await fetch('https://api.example.com/products', {
    cache: 'no-store'
  });
  
  if (!res.ok) {
    throw new Error(`Failed to fetch products: ${res.status}`);
  }
  
  return res.json();
}

export default async function ProductsPage() {
  const products = await getProducts();
  
  return (
    <div className="products-page">
      <h1>Products</h1>
      <div className="products-grid">
        {products.map(product => (
          <div key={product.id} className="product-card">
            <h2>{product.name}</h2>
            <p>Price: ${product.price}</p>
            <p>Stock: {product.stock}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

Now let's see what happens when the API is down:

### Diagnostic Analysis: Server Component Failure

**Browser Behavior**:
The entire page fails to load. Instead of the products page, the user sees Next.js's default error page with a generic error message.

**Browser Console Output**:
```
GET http://localhost:3000/dashboard/products 500 (Internal Server Error)
```

**Terminal Output** (Next.js server):
```bash
Error: Failed to fetch products: 503
    at getProducts (page.tsx:15:11)
    at ProductsPage (page.tsx:21:23)
    
 ⨯ Error: Failed to fetch products: 503
    at getProducts (/app/dashboard/products/page.tsx:15:11)
    at ProductsPage (/app/dashboard/products/page.tsx:21:23)
```

**React DevTools Evidence**:
- Cannot inspect Server Components in React DevTools
- No component tree visible (error occurred during server rendering)
- Page never hydrates because server rendering failed

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: See products page, possibly with loading state or error message
   - Actual: Generic error page, no way to recover without page refresh

2. **What the console reveals**:
   - Key indicator: 500 Internal Server Error from the page route
   - Error location: Server-side, during page rendering
   - The error happened before any HTML was sent to the browser

3. **What the terminal shows**:
   - Component state: Error thrown in `getProducts` function
   - Render behavior: Server rendering failed, no HTML generated
   - The error occurred during server-side data fetching

4. **Root cause identified**: 
   Server Components execute on the server. When they throw errors, those errors occur during server rendering, before any HTML is sent to the browser. Client-side error boundaries can't catch these errors because they never reach the client.

5. **Why the current approach can't solve this**:
   Error boundaries are Client Components. They only catch errors that occur during client-side rendering. Server Component errors happen before the client receives any code.

6. **What we need**:
   Next.js provides special files for handling errors at different levels: `error.tsx` for client errors and `global-error.tsx` for global errors. We need to use these Next.js-specific error handling mechanisms.

## Next.js Error Handling: error.tsx

Next.js provides a file-based convention for error handling. When you create an `error.tsx` file in a route segment, it automatically wraps that segment in an error boundary.

**Key characteristics**:
- `error.tsx` must be a Client Component (marked with `'use client'`)
- It receives `error` and `reset` props automatically
- It catches errors in Server Components, Client Components, and nested layouts
- It creates an error boundary at the route segment level

Let's add error handling to our products page:

```tsx
// src/app/dashboard/products/error.tsx
'use client';

import { useEffect } from 'react';

interface ErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

export default function ProductsError({ error, reset }: ErrorProps) {
  useEffect(() => {
    // Log error to console (we'll add proper logging later)
    console.error('Products page error:', error);
  }, [error]);

  return (
    <div className="error-page">
      <div className="error-content">
        <h1>Failed to Load Products</h1>
        <p className="error-message">
          We couldn't load the products list. This might be a temporary issue.
        </p>
        
        <details className="error-details">
          <summary>Technical details</summary>
          <pre>{error.message}</pre>
          {error.digest && (
            <p className="error-digest">Error ID: {error.digest}</p>
          )}
        </details>

        <div className="error-actions">
          <button onClick={reset} className="btn-primary">
            Try again
          </button>
          <a href="/dashboard" className="btn-secondary">
            Back to Dashboard
          </a>
        </div>
      </div>
    </div>
  );
}
```

### Verification: Next.js Error Handling in Action

Now when the products API fails:

**Browser Behavior**:
- User sees a custom error page instead of the generic Next.js error
- Error message is user-friendly and actionable
- "Try again" button re-attempts the server render
- "Back to Dashboard" provides an escape route

**Browser Console Output**:
```
Products page error: Error: Failed to fetch products: 503
```

**Terminal Output**:
```bash
Error: Failed to fetch products: 503
    at getProducts (page.tsx:15:11)
    at ProductsPage (page.tsx:21:23)
```

**React DevTools Evidence**:
- Error boundary (created by error.tsx) shows in component tree
- Error state is managed by Next.js's error boundary
- Clicking "Try again" triggers a new server render

**Expected vs. Actual improvement**:
- Before: Generic error page, no recovery option
- After: Custom error UI, retry functionality, clear user guidance
- User experience: Professional error handling with recovery path

## Iteration 3: Granular Error Handling with loading.tsx

**Current limitation**: When the products page is loading, there's no loading state. The user sees the previous page until the new page is ready.

**New scenario**: We want to show a loading skeleton while the products are being fetched on the server.

Next.js provides `loading.tsx` for this purpose:

```tsx
// src/app/dashboard/products/loading.tsx
export default function ProductsLoading() {
  return (
    <div className="products-page">
      <h1>Products</h1>
      <div className="products-grid">
        {[...Array(6)].map((_, i) => (
          <div key={i} className="product-card skeleton">
            <div className="skeleton-title" />
            <div className="skeleton-text" />
            <div className="skeleton-text" />
          </div>
        ))}
      </div>
    </div>
  );
}
```

Now let's improve our error handling to work with Suspense boundaries:

```tsx
// src/app/dashboard/products/page.tsx
import { Suspense } from 'react';

interface Product {
  id: string;
  name: string;
  price: number;
  stock: number;
}

async function getProducts(): Promise<Product[]> {
  const res = await fetch('https://api.example.com/products', {
    cache: 'no-store'
  });
  
  if (!res.ok) {
    throw new Error(`Failed to fetch products: ${res.status}`);
  }
  
  return res.json();
}

async function ProductsList() {
  const products = await getProducts();
  
  return (
    <div className="products-grid">
      {products.map(product => (
        <div key={product.id} className="product-card">
          <h2>{product.name}</h2>
          <p>Price: ${product.price}</p>
          <p>Stock: {product.stock}</p>
        </div>
      ))}
    </div>
  );
}

function ProductsLoadingSkeleton() {
  return (
    <div className="products-grid">
      {[...Array(6)].map((_, i) => (
        <div key={i} className="product-card skeleton">
          <div className="skeleton-title" />
          <div className="skeleton-text" />
          <div className="skeleton-text" />
        </div>
      ))}
    </div>
  );
}

export default function ProductsPage() {
  return (
    <div className="products-page">
      <h1>Products</h1>
      <Suspense fallback={<ProductsLoadingSkeleton />}>
        <ProductsList />
      </Suspense>
    </div>
  );
}
```

### Verification: Loading States with Error Handling

**Browser Behavior**:
- Page loads immediately with skeleton UI
- Products list streams in when data is ready
- If error occurs, error boundary catches it
- Page header remains visible during loading and errors

**Expected vs. Actual improvement**:
- Before: Blank screen or previous page during loading
- After: Immediate feedback with skeleton, smooth transition
- User experience: Perceived performance improvement, no jarring transitions

## The Failure: Global Errors Aren't Caught

**Current limitation**: Our `error.tsx` file catches errors in the products route, but what about errors in the root layout or errors that occur before any route is matched?

**New scenario**: The root layout fetches user authentication data. If this fails, the entire app should show an error, not just a single route.

```tsx
// src/app/layout.tsx
import { cookies } from 'next/headers';

async function getUser() {
  const cookieStore = cookies();
  const token = cookieStore.get('auth-token');
  
  if (!token) {
    throw new Error('Not authenticated');
  }
  
  const res = await fetch('https://api.example.com/user', {
    headers: { Authorization: `Bearer ${token.value}` }
  });
  
  if (!res.ok) {
    throw new Error('Failed to fetch user data');
  }
  
  return res.json();
}

export default async function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const user = await getUser();
  
  return (
    <html lang="en">
      <body>
        <header>
          <nav>
            <span>Welcome, {user.name}</span>
          </nav>
        </header>
        <main>{children}</main>
      </body>
    </html>
  );
}
```

### Diagnostic Analysis: Root Layout Failure

**Browser Behavior**:
The entire application fails to load. User sees a generic Next.js error page.

**Terminal Output**:
```bash
Error: Failed to fetch user data
    at getUser (layout.tsx:15:11)
    at RootLayout (layout.tsx:22:17)

 ⨯ Error: Failed to fetch user data
```

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: See login page or error message
   - Actual: Generic error, no way to recover

2. **What the terminal reveals**:
   - Error in root layout, before any route is rendered
   - No route-level error boundary can catch this

3. **Root cause identified**: 
   Errors in the root layout occur before any route-level error boundaries are established. We need a global error handler.

## Global Error Handling: global-error.tsx

For errors that occur in the root layout, Next.js provides `global-error.tsx`:

```tsx
// src/app/global-error.tsx
'use client';

import { useEffect } from 'react';

interface GlobalErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

export default function GlobalError({ error, reset }: GlobalErrorProps) {
  useEffect(() => {
    console.error('Global error:', error);
  }, [error]);

  return (
    <html>
      <body>
        <div className="global-error-page">
          <div className="error-content">
            <h1>Application Error</h1>
            <p>
              We're experiencing technical difficulties. 
              Please try again in a few moments.
            </p>
            
            <details className="error-details">
              <summary>Technical details</summary>
              <pre>{error.message}</pre>
              {error.digest && (
                <p className="error-digest">Error ID: {error.digest}</p>
              )}
            </details>

            <div className="error-actions">
              <button onClick={reset} className="btn-primary">
                Reload Application
              </button>
            </div>
          </div>
        </div>
      </body>
    </html>
  );
}
```

**Important notes about global-error.tsx**:
- It must define its own `<html>` and `<body>` tags (it replaces the root layout)
- It only catches errors in the root layout
- It's rarely triggered in development (use production build to test)
- It's a last resort fallback for catastrophic failures

### Verification: Global Error Handling

**Browser Behavior**:
- User sees custom global error page
- Application structure (html/body) is still valid
- Clear error message and reload option

**Expected vs. Actual improvement**:
- Before: Generic Next.js error, no branding or guidance
- After: Branded error page, clear recovery path
- User experience: Professional error handling even for catastrophic failures

## Error Handling Hierarchy in Next.js

Next.js has a specific hierarchy for error handling:

1. **Component-level**: Try-catch in event handlers and async functions
2. **Route-level**: `error.tsx` catches errors in that route segment
3. **Layout-level**: `error.tsx` in parent segments catches errors in child routes
4. **Global-level**: `global-error.tsx` catches errors in root layout
5. **Unhandled**: Next.js default error page (development) or 500 page (production)

Here's a complete example showing the hierarchy:

```tsx
// Project structure:
// src/app/
// ├── global-error.tsx          ← Catches root layout errors
// ├── error.tsx                 ← Catches errors in app routes
// ├── layout.tsx                ← Root layout
// ├── page.tsx                  ← Home page
// └── dashboard/
//     ├── error.tsx             ← Catches errors in dashboard routes
//     ├── layout.tsx            ← Dashboard layout
//     ├── page.tsx              ← Dashboard home
//     └── products/
//         ├── error.tsx         ← Catches errors in products route
//         ├── loading.tsx       ← Loading state for products
//         └── page.tsx          ← Products page

// src/app/error.tsx
'use client';

export default function AppError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="app-error">
      <h1>Something went wrong</h1>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// src/app/dashboard/error.tsx
'use client';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="dashboard-error">
      <h1>Dashboard Error</h1>
      <p>Failed to load dashboard: {error.message}</p>
      <button onClick={reset}>Retry</button>
      <a href="/">Back to Home</a>
    </div>
  );
}

// src/app/dashboard/products/error.tsx
'use client';

export default function ProductsError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="products-error">
      <h1>Products Error</h1>
      <p>Failed to load products: {error.message}</p>
      <button onClick={reset}>Retry</button>
      <a href="/dashboard">Back to Dashboard</a>
    </div>
  );
}
```

### When to Apply: Error Boundary Placement Strategy

**Route-level error.tsx**:
- What it optimizes for: Granular error handling, specific recovery options
- What it sacrifices: More files to maintain
- When to choose: For routes with specific error handling needs (e.g., payment pages, admin panels)
- When to avoid: For simple routes that can share parent error handling

**Layout-level error.tsx**:
- What it optimizes for: Shared error handling across multiple routes
- What it sacrifices: Less specific error messages
- When to choose: For route groups with similar error handling needs
- When to avoid: When child routes need different error handling strategies

**Global error.tsx**:
- What it optimizes for: Catastrophic failure recovery
- What it sacrifices: Rarely used, hard to test
- When to choose: Always include for production applications
- When to avoid: Never avoid—it's your last line of defense

**Component-level ErrorBoundary**:
- What it optimizes for: Isolated component failures, widget-style UIs
- What it sacrifices: More boilerplate code
- When to choose: For independent components that can fail without affecting siblings
- When to avoid: For tightly coupled components where one failure should affect the group

## Common Failure Modes in Next.js Error Handling

### Symptom: error.tsx doesn't catch Server Component errors

**Browser behavior**:
Generic Next.js error page appears instead of custom error.tsx

**Terminal pattern**:
```bash
Error: Something went wrong
    at ServerComponent (page.tsx:10)
    
 ⨯ Error: Something went wrong
```

**DevTools clues**:
- No custom error UI visible
- Page returns 500 status
- error.tsx file exists but isn't used

**Root cause**: error.tsx file is not marked as a Client Component with `'use client'` directive.

**Solution**: Always add `'use client'` at the top of error.tsx files:

```tsx
// error.tsx must be a Client Component
'use client';

export default function Error({ error, reset }) {
  return <div>Error: {error.message}</div>;
}
```

### Symptom: Reset function doesn't work

**Browser behavior**:
Clicking "Try again" button does nothing

**Console pattern**:
```
(No errors, button click is registered but nothing happens)
```

**DevTools clues**:
- Button onClick fires
- No state changes visible
- Page doesn't re-render

**Root cause**: The reset function only works for errors that occurred during rendering. It doesn't refetch data or reset component state.

**Solution**: For data fetching errors, use router.refresh() or implement custom retry logic:

```tsx
'use client';

import { useRouter } from 'next/navigation';

export default function Error({ error, reset }) {
  const router = useRouter();

  return (
    <div>
      <p>Error: {error.message}</p>
      <button onClick={() => {
        reset(); // Reset error boundary
        router.refresh(); // Refetch server data
      }}>
        Try again
      </button>
    </div>
  );
}
```

### Symptom: Error boundary catches error but page is blank

**Browser behavior**:
Page is completely blank, no error UI visible

**Console pattern**:
```
Error: Something went wrong
(Error boundary should have caught this)
```

**DevTools clues**:
- Error boundary component is mounted
- Error state is set
- Fallback UI is not rendering

**Root cause**: The error boundary's fallback UI itself has an error (e.g., trying to access undefined props).

**Solution**: Keep fallback UI simple and defensive:

```tsx
'use client';

export default function Error({ error, reset }) {
  // Defensive: handle case where error might be undefined
  const message = error?.message || 'An unknown error occurred';
  
  return (
    <div className="error-fallback">
      <h1>Error</h1>
      <p>{message}</p>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

## Logging and monitoring (Sentry)

## The Failure: Console Logs Don't Help in Production

**Current limitation**: We're logging errors to the console, which is fine for development but useless in production. When a user encounters an error, we have no way to know about it unless they report it.

**New scenario**: A user encounters an error in production. They see the error UI and click "Try again," but the error persists. They give up and leave. We never know this happened.

### Diagnostic Analysis: The Invisible Production Error

**Browser Behavior**:
User sees error UI, tries to recover, fails, leaves the site.

**Browser Console Output** (on user's machine):
```
Error: Failed to fetch products: 503
Component stack: ...
```

**Our Monitoring Dashboard**:
(Empty. We have no idea this error occurred.)

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Error is logged and developers are notified
   - Actual: Error happens silently, no one knows

2. **What we're missing**:
   - Error frequency: How often does this happen?
   - User context: What browser, OS, user actions led to this?
   - Error patterns: Is this affecting all users or specific segments?
   - Stack traces: Where exactly is the error occurring?

3. **Root cause identified**: 
   Console logs only exist on the user's machine. We need a centralized error tracking system that captures errors from all users and provides context for debugging.

4. **What we need**:
   A production-grade error monitoring service like Sentry that:
   - Captures errors automatically
   - Provides rich context (browser, OS, user actions)
   - Aggregates errors and shows patterns
   - Alerts us when errors occur
   - Helps us reproduce and fix issues

## Setting Up Sentry

Sentry is the industry standard for error monitoring in React/Next.js applications. Let's integrate it into our dashboard.

First, install the Sentry SDK:

```bash
npm install @sentry/nextjs
```

Initialize Sentry in your project:

```bash
npx @sentry/wizard@latest -i nextjs
```

This wizard creates three configuration files:

```typescript
// sentry.client.config.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  
  // Set tracesSampleRate to 1.0 to capture 100% of transactions for performance monitoring.
  // We recommend adjusting this value in production
  tracesSampleRate: 1.0,
  
  // Capture Replay for 10% of all sessions,
  // plus 100% of sessions with an error
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  
  // Note: if you want to override the automatic release value, do not set a
  // `release` value here - use the environment variable `SENTRY_RELEASE`, so
  // that it will also get attached to your source maps
  
  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,
      blockAllMedia: true,
    }),
  ],
});
```

```typescript
// sentry.server.config.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  
  // Set tracesSampleRate to 1.0 to capture 100% of transactions for performance monitoring.
  // We recommend adjusting this value in production
  tracesSampleRate: 1.0,
  
  // Note: if you want to override the automatic release value, do not set a
  // `release` value here - use the environment variable `SENTRY_RELEASE`, so
  // that it will also get attached to your source maps
});
```

```typescript
// sentry.edge.config.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  
  // Set tracesSampleRate to 1.0 to capture 100% of transactions for performance monitoring.
  // We recommend adjusting this value in production
  tracesSampleRate: 1.0,
});
```

Add your Sentry DSN to environment variables:

```bash
# .env.local
NEXT_PUBLIC_SENTRY_DSN=https://your-dsn@sentry.io/your-project-id
```

Now let's update our error boundaries to send errors to Sentry:

```tsx
// src/components/ErrorBoundary.tsx
'use client';

import React, { Component, ReactNode } from 'react';
import * as Sentry from '@sentry/nextjs';

interface Props {
  children: ReactNode;
  fallback?: (error: Error, reset: () => void) => ReactNode;
  onError?: (error: Error, errorInfo: React.ErrorInfo) => void;
  resetKeys?: Array<string | number>;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = {
      hasError: false,
      error: null
    };
  }

  static getDerivedStateFromError(error: Error): State {
    return {
      hasError: true,
      error
    };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    // Send error to Sentry with component stack
    Sentry.captureException(error, {
      contexts: {
        react: {
          componentStack: errorInfo.componentStack,
        },
      },
    });

    // Call custom error handler if provided
    if (this.props.onError) {
      this.props.onError(error, errorInfo);
    }
  }

  componentDidUpdate(prevProps: Props) {
    if (this.state.hasError && this.props.resetKeys) {
      const prevKeys = prevProps.resetKeys || [];
      const currentKeys = this.props.resetKeys;
      
      const hasChanged = currentKeys.some((key, index) => key !== prevKeys[index]);
      
      if (hasChanged) {
        this.reset();
      }
    }
  }

  reset = () => {
    this.setState({
      hasError: false,
      error: null
    });
  };

  render() {
    if (this.state.hasError && this.state.error) {
      if (this.props.fallback) {
        return this.props.fallback(this.state.error, this.reset);
      }

      return (
        <div className="error-boundary-fallback">
          <h2>Something went wrong</h2>
          <p>{this.state.error.message}</p>
          <button onClick={this.reset}>Try again</button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

Update Next.js error handlers to send errors to Sentry:

```tsx
// src/app/dashboard/products/error.tsx
'use client';

import { useEffect } from 'react';
import * as Sentry from '@sentry/nextjs';

interface ErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

export default function ProductsError({ error, reset }: ErrorProps) {
  useEffect(() => {
    // Send error to Sentry with additional context
    Sentry.captureException(error, {
      tags: {
        section: 'products',
        errorBoundary: 'route-level',
      },
      extra: {
        digest: error.digest,
      },
    });
  }, [error]);

  return (
    <div className="error-page">
      <div className="error-content">
        <h1>Failed to Load Products</h1>
        <p className="error-message">
          We couldn't load the products list. This might be a temporary issue.
        </p>
        
        <details className="error-details">
          <summary>Technical details</summary>
          <pre>{error.message}</pre>
          {error.digest && (
            <p className="error-digest">Error ID: {error.digest}</p>
          )}
        </details>

        <div className="error-actions">
          <button onClick={reset} className="btn-primary">
            Try again
          </button>
          <a href="/dashboard" className="btn-secondary">
            Back to Dashboard
          </a>
        </div>
      </div>
    </div>
  );
}
```

### Verification: Sentry Integration

Now when an error occurs:

**Browser Behavior**:
Same as before—user sees error UI and can retry.

**Sentry Dashboard**:
- New error appears in Sentry dashboard
- Error includes full stack trace
- Component stack shows where error occurred
- Browser, OS, and user context captured
- Error digest/ID for correlation

**Sentry Error Details**:
```
Error: Failed to fetch products: 503

BREADCRUMBS:
  Navigation: /dashboard → /dashboard/products
  XHR: GET /api/products → 503
  
TAGS:
  section: products
  errorBoundary: route-level
  
CONTEXT:
  Browser: Chrome 120.0.0
  OS: macOS 14.0
  URL: https://example.com/dashboard/products
  
STACK TRACE:
  at getProducts (page.tsx:15:11)
  at ProductsPage (page.tsx:21:23)
```

**Expected vs. Actual improvement**:
- Before: Errors invisible in production, no way to track or debug
- After: All errors captured, rich context, aggregation and alerting
- Developer experience: Can proactively fix issues before users report them

## Iteration 4: Adding User Context and Custom Tags

**Current limitation**: Sentry captures errors, but we don't know which user encountered the error or what they were doing.

**New scenario**: We want to know which users are affected by errors and what actions they took before the error occurred.

Let's add user context and custom breadcrumbs:

```tsx
// src/app/layout.tsx
import { cookies } from 'next/headers';
import * as Sentry from '@sentry/nextjs';

async function getUser() {
  const cookieStore = cookies();
  const token = cookieStore.get('auth-token');
  
  if (!token) return null;
  
  const res = await fetch('https://api.example.com/user', {
    headers: { Authorization: `Bearer ${token.value}` }
  });
  
  if (!res.ok) return null;
  
  return res.json();
}

export default async function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const user = await getUser();
  
  // Set user context in Sentry
  if (user) {
    Sentry.setUser({
      id: user.id,
      email: user.email,
      username: user.name,
    });
  }
  
  return (
    <html lang="en">
      <body>
        <header>
          <nav>
            {user && <span>Welcome, {user.name}</span>}
          </nav>
        </header>
        <main>{children}</main>
      </body>
    </html>
  );
}
```

Add custom breadcrumbs for user actions:

```tsx
// src/app/dashboard/products/page.tsx
'use client';

import { useEffect } from 'react';
import * as Sentry from '@sentry/nextjs';

export default function ProductsPage() {
  useEffect(() => {
    // Add breadcrumb when page loads
    Sentry.addBreadcrumb({
      category: 'navigation',
      message: 'User viewed products page',
      level: 'info',
    });
  }, []);

  const handleFilterChange = (filter: string) => {
    // Add breadcrumb for user action
    Sentry.addBreadcrumb({
      category: 'user-action',
      message: `User changed filter to: ${filter}`,
      level: 'info',
      data: { filter },
    });
    
    // Apply filter...
  };

  const handleProductClick = (productId: string) => {
    // Add breadcrumb for user action
    Sentry.addBreadcrumb({
      category: 'user-action',
      message: `User clicked product: ${productId}`,
      level: 'info',
      data: { productId },
    });
    
    // Navigate to product...
  };

  return (
    <div className="products-page">
      {/* UI implementation */}
    </div>
  );
}
```

Add custom tags for better error categorization:

```tsx
// src/app/dashboard/products/error.tsx
'use client';

import { useEffect } from 'react';
import * as Sentry from '@sentry/nextjs';

interface ErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

function categorizeError(error: Error): {
  category: string;
  severity: 'error' | 'warning' | 'fatal';
} {
  const message = error.message.toLowerCase();
  
  if (message.includes('network') || message.includes('fetch')) {
    return { category: 'network', severity: 'error' };
  }
  
  if (message.includes('timeout')) {
    return { category: 'timeout', severity: 'warning' };
  }
  
  if (message.includes('unauthorized') || message.includes('403')) {
    return { category: 'auth', severity: 'error' };
  }
  
  if (message.includes('500') || message.includes('503')) {
    return { category: 'server', severity: 'fatal' };
  }
  
  return { category: 'unknown', severity: 'error' };
}

export default function ProductsError({ error, reset }: ErrorProps) {
  useEffect(() => {
    const { category, severity } = categorizeError(error);
    
    Sentry.captureException(error, {
      level: severity,
      tags: {
        section: 'products',
        errorBoundary: 'route-level',
        errorCategory: category,
      },
      extra: {
        digest: error.digest,
        userAgent: navigator.userAgent,
        timestamp: new Date().toISOString(),
      },
      fingerprint: [
        'products-page',
        category,
        error.message,
      ],
    });
  }, [error]);

  return (
    <div className="error-page">
      {/* Error UI */}
    </div>
  );
}
```

### Verification: Enhanced Error Context

Now when an error occurs:

**Sentry Dashboard**:
```
Error: Failed to fetch products: 503

USER:
  ID: user_123
  Email: john@example.com
  Username: John Doe

BREADCRUMBS:
  [Navigation] User viewed products page
  [User Action] User changed filter to: electronics
  [User Action] User clicked product: prod_456
  [XHR] GET /api/products → 503

TAGS:
  section: products
  errorBoundary: route-level
  errorCategory: server
  
EXTRA:
  digest: abc123
  userAgent: Mozilla/5.0...
  timestamp: 2024-01-15T10:30:00Z

FINGERPRINT:
  products-page
  server
  Failed to fetch products: 503
```

**Expected vs. Actual improvement**:
- Before: Error captured but no user context
- After: Full user journey, categorized errors, custom metadata
- Developer experience: Can reproduce issues and understand user impact

## Iteration 5: Performance Monitoring and Alerts

**Current limitation**: We're capturing errors, but we don't know about performance issues or when errors spike.

**New scenario**: We want to monitor performance metrics and get alerted when error rates increase.

Configure Sentry performance monitoring:

```typescript
// sentry.client.config.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  
  // Performance monitoring
  tracesSampleRate: 0.1, // Sample 10% of transactions in production
  
  // Session replay
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  
  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,
      blockAllMedia: true,
    }),
    Sentry.browserTracingIntegration({
      // Track navigation and page loads
      tracingOrigins: ['localhost', 'example.com', /^\//],
    }),
  ],
  
  // Filter out noise
  beforeSend(event, hint) {
    // Don't send errors from browser extensions
    if (event.exception?.values?.[0]?.stacktrace?.frames?.some(
      frame => frame.filename?.includes('chrome-extension://')
    )) {
      return null;
    }
    
    // Don't send network errors for known flaky endpoints
    if (event.message?.includes('timeout') && 
        event.tags?.endpoint === 'analytics') {
      return null;
    }
    
    return event;
  },
  
  // Environment-specific configuration
  environment: process.env.NODE_ENV,
  release: process.env.NEXT_PUBLIC_VERCEL_GIT_COMMIT_SHA,
});
```

Add custom performance tracking:

```tsx
// src/app/dashboard/products/page.tsx
'use client';

import { useEffect } from 'react';
import * as Sentry from '@sentry/nextjs';

export default function ProductsPage() {
  useEffect(() => {
    // Start performance transaction
    const transaction = Sentry.startTransaction({
      name: 'Products Page Load',
      op: 'page-load',
    });

    // Track data fetching
    const fetchSpan = transaction.startChild({
      op: 'http.client',
      description: 'Fetch products',
    });

    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        fetchSpan.finish();
        
        // Track rendering
        const renderSpan = transaction.startChild({
          op: 'react.render',
          description: 'Render products list',
        });
        
        // Simulate render time
        setTimeout(() => {
          renderSpan.finish();
          transaction.finish();
        }, 0);
      })
      .catch(error => {
        fetchSpan.finish();
        transaction.finish();
        throw error;
      });
  }, []);

  return (
    <div className="products-page">
      {/* UI implementation */}
    </div>
  );
}
```

Configure Sentry alerts in the Sentry dashboard:

1. **Error Rate Alert**:
   - Condition: Error rate > 5% of sessions
   - Action: Email team, post to Slack
   - Frequency: Maximum once per hour

2. **New Error Alert**:
   - Condition: First occurrence of new error
   - Action: Email on-call engineer
   - Frequency: Immediate

3. **Performance Alert**:
   - Condition: P95 page load time > 3 seconds
   - Action: Post to Slack #performance channel
   - Frequency: Daily digest

### Verification: Complete Monitoring Setup

**Sentry Dashboard Features**:
- Error tracking with full context
- Performance monitoring (page loads, API calls)
- Session replay (watch user sessions that had errors)
- Release tracking (correlate errors with deployments)
- Alerts (get notified when things go wrong)

**Expected vs. Actual improvement**:
- Before: Reactive debugging after user reports
- After: Proactive monitoring and alerting
- Developer experience: Fix issues before they impact many users

## When to Apply: Error Monitoring Strategy

**Sentry (or similar service)**:
- What it optimizes for: Production error visibility, user impact analysis
- What it sacrifices: Additional service cost, some performance overhead
- When to choose: Always for production applications
- When to avoid: Never avoid—it's essential for production

**Error sampling rate**:
- What it optimizes for: Cost reduction, performance
- What it sacrifices: Some errors might not be captured
- When to choose: High-traffic applications (sample 10-20%)
- When to avoid: Low-traffic applications (sample 100%)

**Session replay**:
- What it optimizes for: Understanding user context
- What it sacrifices: Privacy concerns, bandwidth
- When to choose: Complex UIs where user actions matter
- When to avoid: Privacy-sensitive applications, high-traffic sites

**Performance monitoring**:
- What it optimizes for: Identifying slow pages and API calls
- What it sacrifices: Additional overhead, cost
- When to choose: User-facing applications where performance matters
- When to avoid: Internal tools with low traffic

## Common Failure Modes in Error Monitoring

### Symptom: Errors not appearing in Sentry

**Browser behavior**:
Errors occur and are logged to console, but don't appear in Sentry dashboard.

**Console pattern**:
```
Error: Something went wrong
(No Sentry-related logs)
```

**DevTools clues**:
- Network tab shows no requests to Sentry
- Sentry SDK might not be initialized
- DSN might be missing or incorrect

**Root cause**: Sentry not properly initialized or DSN not configured.

**Solution**: Verify Sentry initialization and environment variables:

```bash
# Check environment variables
echo $NEXT_PUBLIC_SENTRY_DSN

# Verify Sentry is initialized
# Add this to your app to test:
```

```typescript
// Test Sentry initialization
import * as Sentry from '@sentry/nextjs';

// This should send a test error to Sentry
Sentry.captureMessage('Test message from app');
```

### Symptom: Too many errors flooding Sentry

**Sentry Dashboard**:
Thousands of identical errors, quota exceeded warnings.

**Root cause**: Error loop or missing error deduplication.

**Solution**: Implement error fingerprinting and rate limiting:

```typescript
// sentry.client.config.ts
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  
  beforeSend(event, hint) {
    // Deduplicate errors by fingerprint
    if (event.exception?.values?.[0]) {
      const error = event.exception.values[0];
      event.fingerprint = [
        error.type || 'Error',
        error.value || 'Unknown',
        // Include first frame of stack trace for uniqueness
        error.stacktrace?.frames?.[0]?.filename || 'unknown',
      ];
    }
    
    return event;
  },
  
  // Rate limit errors
  maxBreadcrumbs: 50,
  
  // Filter out known noisy errors
  ignoreErrors: [
    'ResizeObserver loop limit exceeded',
    'Non-Error promise rejection captured',
  ],
});
```

### Symptom: Sensitive data leaked to Sentry

**Sentry Dashboard**:
Error context includes passwords, API keys, or PII.

**Root cause**: Not scrubbing sensitive data before sending to Sentry.

**Solution**: Implement data scrubbing:

```typescript
// sentry.client.config.ts
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  
  beforeSend(event, hint) {
    // Remove sensitive data from event
    if (event.request?.headers) {
      delete event.request.headers['Authorization'];
      delete event.request.headers['Cookie'];
    }
    
    // Scrub sensitive data from extra context
    if (event.extra) {
      Object.keys(event.extra).forEach(key => {
        if (key.toLowerCase().includes('password') ||
            key.toLowerCase().includes('token') ||
            key.toLowerCase().includes('secret')) {
          event.extra![key] = '[Filtered]';
        }
      });
    }
    
    return event;
  },
  
  // Use session replay with masking
  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,
      maskAllInputs: true,
      blockAllMedia: true,
    }),
  ],
});
```

## User-friendly error states

## The Failure: Technical Error Messages Confuse Users

**Current limitation**: Our error messages are developer-focused. They show technical details that confuse non-technical users and don't provide clear guidance on what to do next.

**New scenario**: A user encounters a network error. They see "Failed to fetch products: 503" and don't know what that means or what to do about it.

### Diagnostic Analysis: Poor Error UX

**Browser Behavior**:
User sees technical error message: "Failed to fetch products: 503"

**User's Mental Model**:
- "What's a 503?"
- "Is this my fault?"
- "Should I wait or try again?"
- "Is my data safe?"

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Clear explanation of what went wrong and what to do
   - Actual: Technical jargon, no actionable guidance

2. **What's missing**:
   - Plain language explanation
   - Clear next steps
   - Indication of severity (temporary vs. permanent)
   - Reassurance about data safety

3. **Root cause identified**: 
   We're showing raw error messages meant for developers, not user-friendly explanations.

4. **What we need**:
   A system for translating technical errors into user-friendly messages with clear guidance.

## Creating User-Friendly Error Messages

Let's build a comprehensive error message system:

```typescript
// src/lib/errors/errorMessages.ts

export interface ErrorMessage {
  title: string;
  description: string;
  action: string;
  severity: 'info' | 'warning' | 'error' | 'critical';
  canRetry: boolean;
  showTechnicalDetails: boolean;
}

export function getUserFriendlyError(error: Error): ErrorMessage {
  const message = error.message.toLowerCase();
  
  // Network errors
  if (message.includes('failed to fetch') || message.includes('network')) {
    return {
      title: 'Connection Problem',
      description: 'We're having trouble connecting to our servers. This is usually temporary.',
      action: 'Check your internet connection and try again in a moment.',
      severity: 'warning',
      canRetry: true,
      showTechnicalDetails: false,
    };
  }
  
  // Timeout errors
  if (message.includes('timeout')) {
    return {
      title: 'Request Timed Out',
      description: 'The request took too long to complete. Our servers might be busy.',
      action: 'Please try again. If this keeps happening, try again later.',
      severity: 'warning',
      canRetry: true,
      showTechnicalDetails: false,
    };
  }
  
  // Server errors (5xx)
  if (message.includes('500') || message.includes('503') || message.includes('502')) {
    return {
      title: 'Server Error',
      description: 'Our servers are experiencing issues. We've been notified and are working on it.',
      action: 'Please try again in a few minutes.',
      severity: 'error',
      canRetry: true,
      showTechnicalDetails: false,
    };
  }
  
  // Authentication errors
  if (message.includes('unauthorized') || message.includes('401') || message.includes('403')) {
    return {
      title: 'Access Denied',
      description: 'You don't have permission to access this content, or your session has expired.',
      action: 'Please sign in again.',
      severity: 'warning',
      canRetry: false,
      showTechnicalDetails: false,
    };
  }
  
  // Not found errors
  if (message.includes('404') || message.includes('not found')) {
    return {
      title: 'Content Not Found',
      description: 'The content you're looking for doesn't exist or has been moved.',
      action: 'Check the URL or go back to the previous page.',
      severity: 'info',
      canRetry: false,
      showTechnicalDetails: false,
    };
  }
  
  // Validation errors
  if (message.includes('validation') || message.includes('invalid')) {
    return {
      title: 'Invalid Input',
      description: 'Some of the information provided is incorrect or incomplete.',
      action: 'Please check your input and try again.',
      severity: 'warning',
      canRetry: true,
      showTechnicalDetails: false,
    };
  }
  
  // Rate limiting
  if (message.includes('rate limit') || message.includes('too many requests')) {
    return {
      title: 'Too Many Requests',
      description: 'You've made too many requests in a short time.',
      action: 'Please wait a moment before trying again.',
      severity: 'warning',
      canRetry: true,
      showTechnicalDetails: false,
    };
  }
  
  // Default for unknown errors
  return {
    title: 'Something Went Wrong',
    description: 'An unexpected error occurred. We've been notified and will look into it.',
    action: 'Please try again. If the problem persists, contact support.',
    severity: 'error',
    canRetry: true,
    showTechnicalDetails: true,
  };
}
```

Now let's create a reusable error display component:

```tsx
// src/components/ErrorDisplay.tsx
'use client';

import { useState } from 'react';
import { getUserFriendlyError, ErrorMessage } from '@/lib/errors/errorMessages';

interface ErrorDisplayProps {
  error: Error;
  onRetry?: () => void;
  onDismiss?: () => void;
  className?: string;
}

export function ErrorDisplay({ 
  error, 
  onRetry, 
  onDismiss,
  className = '' 
}: ErrorDisplayProps) {
  const [showDetails, setShowDetails] = useState(false);
  const errorMessage = getUserFriendlyError(error);
  
  const severityStyles = {
    info: 'bg-blue-50 border-blue-200 text-blue-900',
    warning: 'bg-yellow-50 border-yellow-200 text-yellow-900',
    error: 'bg-red-50 border-red-200 text-red-900',
    critical: 'bg-red-100 border-red-300 text-red-950',
  };
  
  const severityIcons = {
    info: 'ℹ️',
    warning: '⚠️',
    error: '❌',
    critical: '🚨',
  };

  return (
    <div 
      className={`error-display border-2 rounded-lg p-6 ${severityStyles[errorMessage.severity]} ${className}`}
      role="alert"
      aria-live="assertive"
    >
      <div className="flex items-start gap-4">
        <span className="text-2xl" aria-hidden="true">
          {severityIcons[errorMessage.severity]}
        </span>
        
        <div className="flex-1">
          <h2 className="text-lg font-semibold mb-2">
            {errorMessage.title}
          </h2>
          
          <p className="mb-3">
            {errorMessage.description}
          </p>
          
          <p className="text-sm font-medium mb-4">
            {errorMessage.action}
          </p>
          
          {errorMessage.showTechnicalDetails && (
            <details className="mb-4">
              <summary 
                className="cursor-pointer text-sm font-medium hover:underline"
                onClick={() => setShowDetails(!showDetails)}
              >
                {showDetails ? 'Hide' : 'Show'} technical details
              </summary>
              <div className="mt-2 p-3 bg-white bg-opacity-50 rounded border">
                
                  {error.message}
```

Update our error boundaries to use the new error display:

```tsx
// src/app/dashboard/products/error.tsx
'use client';

import { useEffect } from 'react';
import { useRouter } from 'next/navigation';
import * as Sentry from '@sentry/nextjs';
import { ErrorDisplay } from '@/components/ErrorDisplay';

interface ErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

export default function ProductsError({ error, reset }: ErrorProps) {
  const router = useRouter();
  
  useEffect(() => {
    Sentry.captureException(error, {
      tags: {
        section: 'products',
        errorBoundary: 'route-level',
      },
      extra: {
        digest: error.digest,
      },
    });
  }, [error]);

  const handleRetry = () => {
    reset();
    router.refresh();
  };

  const handleDismiss = () => {
    router.push('/dashboard');
  };

  return (
    <div className="products-error-page p-6">
      <ErrorDisplay
        error={error}
        onRetry={handleRetry}
        onDismiss={handleDismiss}
      />
    </div>
  );
}
```

### Verification: User-Friendly Error Messages

Now when a network error occurs:

**Browser Behavior**:
- User sees: "Connection Problem"
- Description: "We're having trouble connecting to our servers. This is usually temporary."
- Action: "Check your internet connection and try again in a moment."
- Clear "Try Again" button
- Technical details hidden by default

**User's Mental Model**:
- "Oh, it's a connection issue"
- "It's probably temporary"
- "I should check my internet"
- "I can try again"

**Expected vs. Actual improvement**:
- Before: "Failed to fetch products: 503" (confusing)
- After: Clear explanation, actionable guidance, appropriate severity
- User experience: Reduced confusion, clear next steps, maintained trust

## Iteration 6: Contextual Error States

**Current limitation**: All errors show the same generic error display, regardless of where they occur or what the user was doing.

**New scenario**: We want different error presentations for different contexts: inline errors for form fields, toast notifications for background operations, full-page errors for critical failures.

Let's create context-aware error components:

```tsx
// src/components/errors/InlineError.tsx
'use client';

import { getUserFriendlyError } from '@/lib/errors/errorMessages';

interface InlineErrorProps {
  error: Error;
  onRetry?: () => void;
  compact?: boolean;
}

export function InlineError({ error, onRetry, compact = false }: InlineErrorProps) {
  const errorMessage = getUserFriendlyError(error);
  
  if (compact) {
    return (
      <div className="inline-error-compact flex items-center gap-2 text-sm text-red-600">
        <span>❌</span>
        <span>{errorMessage.title}</span>
        {onRetry && errorMessage.canRetry && (
          <button
            onClick={onRetry}
            className="text-xs underline hover:no-underline"
          >
            Retry
          </button>
        )}
      </div>
    );
  }

  return (
    <div className="inline-error bg-red-50 border border-red-200 rounded p-4">
      <div className="flex items-start gap-3">
        <span className="text-xl">❌</span>
        <div className="flex-1">
          <p className="font-medium text-red-900">{errorMessage.title}</p>
          <p className="text-sm text-red-700 mt-1">{errorMessage.description}</p>
          {onRetry && errorMessage.canRetry && (
            <button
              onClick={onRetry}
              className="mt-2 text-sm font-medium text-red-900 underline hover:no-underline"
            >
              Try again
            </button>
          )}
        </div>
      </div>
    </div>
  );
}
```

```tsx
// src/components/errors/ToastError.tsx
'use client';

import { useEffect, useState } from 'react';
import { getUserFriendlyError } from '@/lib/errors/errorMessages';

interface ToastErrorProps {
  error: Error;
  onDismiss: () => void;
  duration?: number;
}

export function ToastError({ error, onDismiss, duration = 5000 }: ToastErrorProps) {
  const [isVisible, setIsVisible] = useState(true);
  const errorMessage = getUserFriendlyError(error);

  useEffect(() => {
    const timer = setTimeout(() => {
      setIsVisible(false);
      setTimeout(onDismiss, 300); // Wait for fade out animation
    }, duration);

    return () => clearTimeout(timer);
  }, [duration, onDismiss]);

  if (!isVisible) return null;

  return (
    <div 
      className={`toast-error fixed bottom-4 right-4 max-w-md bg-white border-2 border-red-200 rounded-lg shadow-lg p-4 transition-opacity duration-300 ${isVisible ? 'opacity-100' : 'opacity-0'}`}
      role="alert"
      aria-live="polite"
    >
      <div className="flex items-start gap-3">
        <span className="text-xl">❌</span>
        <div className="flex-1">
          <p className="font-semibold text-gray-900">{errorMessage.title}</p>
          <p className="text-sm text-gray-600 mt-1">{errorMessage.description}</p>
        </div>
        <button
          onClick={() => {
            setIsVisible(false);
            setTimeout(onDismiss, 300);
          }}
          className="text-gray-400 hover:text-gray-600"
          aria-label="Dismiss"
        >
          ✕
        </button>
      </div>
    </div>
  );
}
```

```tsx
// src/components/errors/EmptyState.tsx
'use client';

import { getUserFriendlyError } from '@/lib/errors/errorMessages';

interface EmptyStateProps {
  error: Error;
  onRetry?: () => void;
  illustration?: React.ReactNode;
}

export function EmptyState({ error, onRetry, illustration }: EmptyStateProps) {
  const errorMessage = getUserFriendlyError(error);

  return (
    <div className="empty-state flex flex-col items-center justify-center min-h-[400px] p-8 text-center">
      {illustration || (
        <div className="text-6xl mb-4" aria-hidden="true">
          📭
        </div>
      )}
      
      <h2 className="text-2xl font-semibold text-gray-900 mb-2">
        {errorMessage.title}
      </h2>
      
      <p className="text-gray-600 max-w-md mb-2">
        {errorMessage.description}
      </p>
      
      <p className="text-sm text-gray-500 mb-6">
        {errorMessage.action}
      </p>
      
      {onRetry && errorMessage.canRetry && (
        <button
          onClick={onRetry}
          className="px-6 py-3 bg-blue-600 text-white rounded-lg font-medium hover:bg-blue-700 transition-colors"
        >
          Try Again
        </button>
      )}
    </div>
  );
}
```

Now let's use these contextual error components in different scenarios:

```tsx
// src/app/dashboard/products/page.tsx
'use client';

import { useState, useEffect } from 'react';
import { InlineError } from '@/components/errors/InlineError';
import { ToastError } from '@/components/errors/ToastError';
import { EmptyState } from '@/components/errors/EmptyState';

interface Product {
  id: string;
  name: string;
  price: number;
  stock: number;
}

export default function ProductsPage() {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  const [toastError, setToastError] = useState<Error | null>(null);

  const fetchProducts = async () => {
    try {
      setLoading(true);
      setError(null);
      
      const res = await fetch('/api/products');
      if (!res.ok) {
        throw new Error(`Failed to fetch products: ${res.status}`);
      }
      
      const data = await res.json();
      setProducts(data);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchProducts();
  }, []);

  const handleDelete = async (productId: string) => {
    try {
      const res = await fetch(`/api/products/${productId}`, {
        method: 'DELETE',
      });
      
      if (!res.ok) {
        throw new Error('Failed to delete product');
      }
      
      setProducts(prev => prev.filter(p => p.id !== productId));
    } catch (err) {
      // Show toast for background operation errors
      setToastError(err as Error);
    }
  };

  if (loading) {
    return <div className="p-6">Loading products...</div>;
  }

  // Full-page error for critical failures
  if (error && products.length === 0) {
    return (
      <div className="p-6">
        <EmptyState
          error={error}
          onRetry={fetchProducts}
        />
      </div>
    );
  }

  return (
    <div className="products-page p-6">
      <div className="flex items-center justify-between mb-6">
        <h1 className="text-2xl font-bold">Products</h1>
        <button className="px-4 py-2 bg-blue-600 text-white rounded">
          Add Product
        </button>
      </div>

      {/* Inline error for non-critical failures */}
      {error && products.length > 0 && (
        <InlineError
          error={error}
          onRetry={fetchProducts}
        />
      )}

      <div className="products-grid grid grid-cols-3 gap-4 mt-4">
        {products.map(product => (
          <div key={product.id} className="product-card border rounded p-4">
            <h2 className="font-semibold">{product.name}</h2>
            <p className="text-gray-600">${product.price}</p>
            <p className="text-sm text-gray-500">Stock: {product.stock}</p>
            <button
              onClick={() => handleDelete(product.id)}
              className="mt-2 text-sm text-red-600 hover:underline"
            >
              Delete
            </button>
          </div>
        ))}
      </div>

      {/* Toast for background operation errors */}
      {toastError && (
        <ToastError
          error={toastError}
          onDismiss={() => setToastError(null)}
        />
      )}
    </div>
  );
}
```

### Verification: Contextual Error States

**Scenario 1: Initial load fails**
- User sees: Empty state with illustration
- Message: User-friendly explanation
- Action: Large "Try Again" button

**Scenario 2: Refresh fails but data exists**
- User sees: Inline error banner at top
- Message: Brief error message
- Action: Small "Retry" link
- Products: Still visible below

**Scenario 3: Delete operation fails**
- User sees: Toast notification in bottom-right
- Message: Brief error message
- Action: Auto-dismisses after 5 seconds
- Products: Remain unchanged

**Expected vs. Actual improvement**:
- Before: Same error display for all contexts
- After: Appropriate error presentation based on context and severity
- User experience: Less disruptive, more contextually appropriate

## Iteration 7: Accessibility and Internationalization

**Current limitation**: Our error messages aren't accessible to screen reader users and only support English.

**New scenario**: We want our error handling to be accessible and support multiple languages.

Let's add accessibility features:

```tsx
// src/components/ErrorDisplay.tsx (updated)
'use client';

import { useState, useRef, useEffect } from 'react';
import { getUserFriendlyError } from '@/lib/errors/errorMessages';

interface ErrorDisplayProps {
  error: Error;
  onRetry?: () => void;
  onDismiss?: () => void;
  className?: string;
}

export function ErrorDisplay({ 
  error, 
  onRetry, 
  onDismiss,
  className = '' 
}: ErrorDisplayProps) {
  const [showDetails, setShowDetails] = useState(false);
  const errorMessage = getUserFriendlyError(error);
  const retryButtonRef = useRef<HTMLButtonElement>(null);
  
  // Focus retry button when error appears
  useEffect(() => {
    if (errorMessage.canRetry && retryButtonRef.current) {
      retryButtonRef.current.focus();
    }
  }, [errorMessage.canRetry]);
  
  const severityStyles = {
    info: 'bg-blue-50 border-blue-200 text-blue-900',
    warning: 'bg-yellow-50 border-yellow-200 text-yellow-900',
    error: 'bg-red-50 border-red-200 text-red-900',
    critical: 'bg-red-100 border-red-300 text-red-950',
  };
  
  const severityIcons = {
    info: 'ℹ️',
    warning: '⚠️',
    error: '❌',
    critical: '🚨',
  };
  
  const severityLabels = {
    info: 'Information',
    warning: 'Warning',
    error: 'Error',
    critical: 'Critical Error',
  };

  return (
    <div 
      className={`error-display border-2 rounded-lg p-6 ${severityStyles[errorMessage.severity]} ${className}`}
      role="alert"
      aria-live="assertive"
      aria-atomic="true"
    >
      <div className="flex items-start gap-4">
        <span 
          className="text-2xl" 
          aria-label={severityLabels[errorMessage.severity]}
          role="img"
        >
          {severityIcons[errorMessage.severity]}
        </span>
        
        <div className="flex-1">
          <h2 
            className="text-lg font-semibold mb-2"
            id="error-title"
          >
            {errorMessage.title}
          </h2>
          
          <p 
            className="mb-3"
            id="error-description"
          >
            {errorMessage.description}
          </p>
          
          <p 
            className="text-sm font-medium mb-4"
            id="error-action"
          >
            {errorMessage.action}
          </p>
          
          {errorMessage.showTechnicalDetails && (
            <details className="mb-4">
              <summary 
                className="cursor-pointer text-sm font-medium hover:underline"
                onClick={() => setShowDetails(!showDetails)}
                aria-expanded={showDetails}
                aria-controls="technical-details"
              >
                {showDetails ? 'Hide' : 'Show'} technical details
              </summary>
              <div 
                id="technical-details"
                className="mt-2 p-3 bg-white bg-opacity-50 rounded border"
                role="region"
                aria-label="Technical error details"
              >
                
                  {error.message}
```

Add internationalization support:

```typescript
// src/lib/errors/errorMessages.ts (updated)

export interface ErrorMessage {
  title: string;
  description: string;
  action: string;
  severity: 'info' | 'warning' | 'error' | 'critical';
  canRetry: boolean;
  showTechnicalDetails: boolean;
}

interface ErrorTranslations {
  [key: string]: {
    title: string;
    description: string;
    action: string;
  };
}

const translations: { [locale: string]: ErrorTranslations } = {
  en: {
    network: {
      title: 'Connection Problem',
      description: 'We're having trouble connecting to our servers. This is usually temporary.',
      action: 'Check your internet connection and try again in a moment.',
    },
    timeout: {
      title: 'Request Timed Out',
      description: 'The request took too long to complete. Our servers might be busy.',
      action: 'Please try again. If this keeps happening, try again later.',
    },
    server: {
      title: 'Server Error',
      description: 'Our servers are experiencing issues. We've been notified and are working on it.',
      action: 'Please try again in a few minutes.',
    },
    // ... other error types
  },
  es: {
    network: {
      title: 'Problema de Conexión',
      description: 'Tenemos problemas para conectarnos a nuestros servidores. Esto suele ser temporal.',
      action: 'Verifica tu conexión a internet e intenta nuevamente en un momento.',
    },
    timeout: {
      title: 'Tiempo de Espera Agotado',
      description: 'La solicitud tardó demasiado en completarse. Nuestros servidores podrían estar ocupados.',
      action: 'Por favor, intenta nuevamente. Si esto continúa, intenta más tarde.',
    },
    server: {
      title: 'Error del Servidor',
      description: 'Nuestros servidores están experimentando problemas. Hemos sido notificados y estamos trabajando en ello.',
      action: 'Por favor, intenta nuevamente en unos minutos.',
    },
    // ... other error types
  },
};

export function getUserFriendlyError(
  error: Error, 
  locale: string = 'en'
): ErrorMessage {
  const message = error.message.toLowerCase();
  const localeTranslations = translations[locale] || translations.en;
  
  // Determine error type
  let errorType = 'unknown';
  if (message.includes('failed to fetch') || message.includes('network')) {
    errorType = 'network';
  } else if (message.includes('timeout')) {
    errorType = 'timeout';
  } else if (message.includes('500') || message.includes('503') || message.includes('502')) {
    errorType = 'server';
  }
  // ... other error type checks
  
  const translation = localeTranslations[errorType] || localeTranslations.unknown;
  
  return {
    title: translation.title,
    description: translation.description,
    action: translation.action,
    severity: errorType === 'server' ? 'error' : 'warning',
    canRetry: errorType !== 'auth',
    showTechnicalDetails: errorType === 'unknown',
  };
}
```

### Verification: Accessible and Internationalized Errors

**Accessibility Features**:
- Screen reader announces error with `role="alert"` and `aria-live="assertive"`
- Retry button receives focus automatically
- All interactive elements have proper ARIA labels
- Keyboard navigation works correctly
- Focus management guides user to recovery action

**Internationalization**:
- Error messages display in user's preferred language
- Fallback to English if translation not available
- Technical details remain in English (for support purposes)

**Expected vs. Actual improvement**:
- Before: Not accessible, English only
- After: Fully accessible, multi-language support
- User experience: Inclusive, works for all users

## The Complete Journey: From Crash to Professional Error Handling

| Iteration | Failure Mode                      | Technique Applied                | Result                          | User Impact                    |
| --------- | --------------------------------- | -------------------------------- | ------------------------------- | ------------------------------ |
| 0         | Entire app crashes on error       | None                             | Blank screen                    | Complete failure               |
| 1         | App crashes, no recovery          | Error Boundary                   | Isolated failures               | Partial functionality          |
| 2         | Generic errors, no retry          | Granular boundaries + reset      | Specific errors, retry works    | Clear recovery path            |
| 3         | Server errors not caught          | Next.js error.tsx                | Server errors handled           | Consistent error handling      |
| 4         | Errors invisible in production    | Sentry integration               | All errors tracked              | Proactive issue resolution     |
| 5         | Technical messages confuse users  | User-friendly error messages     | Clear, actionable guidance      | Reduced confusion              |
| 6         | Same error UI everywhere          | Contextual error components      | Appropriate error presentation  | Less disruptive                |
| 7         | Not accessible, English only      | A11y + i18n                      | Accessible, multi-language      | Inclusive experience           |

### Final Implementation: Production-Ready Error Handling

Our complete error handling system now includes:

1. **Error Boundaries**: Catch rendering errors at appropriate levels
2. **Next.js Error Files**: Handle server-side errors with error.tsx and global-error.tsx
3. **Sentry Integration**: Track all errors with rich context
4. **User-Friendly Messages**: Translate technical errors into clear guidance
5. **Contextual Presentation**: Show errors appropriately based on context
6. **Accessibility**: Full screen reader support and keyboard navigation
7. **Internationalization**: Multi-language error messages

### Decision Framework: Error Handling Strategy

**When to use Error Boundaries**:
- Isolate independent components (widgets, cards)
- Prevent cascading failures
- Provide component-specific recovery

**When to use error.tsx**:
- Handle route-level errors
- Provide page-specific error UI
- Catch Server Component errors

**When to use global-error.tsx**:
- Handle root layout errors
- Provide last-resort fallback
- Maintain branding during catastrophic failures

**When to use inline errors**:
- Form validation errors
- Non-critical failures
- Errors that don't block main content

**When to use toast notifications**:
- Background operation failures
- Non-blocking errors
- Temporary status updates

**When to use empty states**:
- Critical failures with no data
- First-time load failures
- Complete feature unavailability

### Lessons Learned

1. **Error boundaries are essential**: Never ship a React app without error boundaries
2. **User-friendly messages matter**: Technical errors confuse users; translate them
3. **Context determines presentation**: Different errors need different UI treatments
4. **Monitoring is not optional**: You can't fix errors you don't know about
5. **Accessibility is fundamental**: Error handling must work for all users
6. **Recovery paths are critical**: Always provide a way forward
7. **Test error states**: Error handling is often the least-tested code path
