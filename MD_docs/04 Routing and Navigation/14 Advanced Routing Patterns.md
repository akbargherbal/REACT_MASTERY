# Chapter 14: Advanced Routing Patterns

## Code splitting and lazy loading

## The Monolithic Monster: Why Loading Everything at Once Fails

In modern web applications, speed is not a feature; it's a prerequisite. Users expect pages to load instantly. One of the biggest enemies of a fast initial load is a large JavaScript bundle. When your application grows, you add more pages, more features, and more libraries. Without a deliberate strategy, all of this code gets bundled into a single, massive file that every user must download when they first visit your site, regardless of which page they actually need.

This chapter will guide you through transforming a slow, monolithic application into a highly optimized, performant user experience using advanced routing patterns available in Next.js.

### Phase 1: Establish the Reference Implementation

We will build a simple E-commerce Admin Dashboard. This dashboard will have three main pages:
1.  **Overview**: The main landing page.
2.  **Orders**: A page to view recent orders, containing a complex charting library.
3.  **Products**: A page to manage a long list of products.

This dashboard is our **anchor example**. We will start with a naive implementation and progressively refine it throughout this chapter, demonstrating how each advanced routing pattern solves a specific, tangible problem.

First, let's set up the initial project structure.

**Project Structure**:
```
src/
├── app/
│   ├── layout.tsx         # Root layout with navigation
│   ├── page.tsx           # Overview page
│   ├── orders/
│   │   └── page.tsx       # Orders page (with heavy component)
│   └── products/
│       └── page.tsx       # Products page
└── components/
    ├── Nav.tsx            # Navigation component
    └── HeavyChart.tsx     # A simulated heavy component for the Orders page
```

Here is the initial, problematic code.

```tsx
// src/components/HeavyChart.tsx
// A placeholder for a large charting library like D3 or Chart.js
// Imagine this component is 500KB of JavaScript.
"use client";

export default function HeavyChart() {
  return (
    <div className="bg-gray-100 p-8 rounded-lg">
      <h2 className="text-xl font-bold mb-4">Sales Data Chart</h2>
      <p>(This is a placeholder for a very large and complex chart component.)</p>
      <div className="w-full h-64 bg-gray-300 animate-pulse mt-4"></div>
    </div>
  );
}
```

```tsx
// src/app/orders/page.tsx
import HeavyChart from "@/components/HeavyChart";

export default function OrdersPage() {
  return (
    <div>
      <h1 className="text-3xl font-bold mb-6">Orders</h1>
      <HeavyChart />
    </div>
  );
}
```

```tsx
// src/app/products/page.tsx
export default function ProductsPage() {
  // Generate a long list to demonstrate scrolling issues later
  const products = Array.from({ length: 100 }, (_, i) => `Product ${i + 1}`);

  return (
    <div>
      <h1 className="text-3xl font-bold mb-6">Products</h1>
      <ul className="space-y-2">
        {products.map((product) => (
          <li key={product} className="p-2 border rounded">
            {product}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

```tsx
// src/components/Nav.tsx
import Link from "next/link";

export default function Nav() {
  return (
    <nav className="bg-gray-800 p-4">
      <ul className="flex space-x-6 text-white">
        <li><Link href="/" className="hover:text-blue-300">Overview</Link></li>
        <li><Link href="/orders" className="hover:text-blue-300">Orders</Link></li>
        <li><Link href="/products" className="hover:text-blue-300">Products</Link></li>
      </ul>
    </nav>
  );
}
```

```tsx
// src/app/layout.tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import Nav from "@/components/Nav";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "Admin Dashboard",
  description: "E-commerce Admin",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <Nav />
        <main className="p-8">{children}</main>
      </body>
    </html>
  );
}
```

```tsx
// src/app/page.tsx
export default function OverviewPage() {
  return (
    <div>
      <h1 className="text-3xl font-bold mb-6">Dashboard Overview</h1>
      <p>Welcome to your admin dashboard.</p>
    </div>
  );
}
```

### Iteration 0: The Problem of the Monolithic Bundle

With the code above, we have a fully functional application. However, it hides a severe performance problem. Let's run the app (`npm run dev`), open the browser's developer tools, and analyze what happens on the initial page load.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The user visits `http://localhost:3000`. The page loads, but on a slow connection, there's a noticeable delay before anything appears.

**Browser DevTools - Network Tab** (Filter by JS, Disable cache):
When you load the homepage (`/`), you'll see a large JavaScript file being downloaded. This file is often named something like `app/page-....js` or a similar hash-based name.

```
Name                    Status  Type      Size
----------------------------------------------------
page-....js             200     script    525 KB  <-- PROBLEM
...other chunks
```

**Performance Metrics** (Simulated Slow 3G connection):
- **Bundle size**: ~525 KB (where 500 KB is our simulated `HeavyChart`)
- **Initial load time**: 3.5s
- **Largest Contentful Paint (LCP)**: 3.2s
- **Time to Interactive (TTI)**: 3.4s

**Let's parse this evidence**:

1.  **What the user experiences**: A slow initial page load. They are forced to wait for code they might never use.

    -   **Expected**: The homepage should load quickly, only downloading the code necessary for the "Overview" page.
    -   **Actual**: The browser downloads the code for the Overview, Orders, *and* Products pages, including the massive `HeavyChart` component, just to show the simple homepage.

2.  **What the Network tab reveals**: A single, large JavaScript chunk contains the code for all our pages.

    -   **Key indicator**: The size of the initial JS chunk is far larger than what's needed for the simple overview page.

3.  **Root cause identified**: By default, Next.js bundles components that are statically imported together. Because our `orders/page.tsx` statically imports `HeavyChart`, and it's part of the same application, its code is included in the initial JavaScript payload.

4.  **Why the current approach can't solve this**: Static `import` statements are resolved at build time. The bundler sees `import HeavyChart from '@/components/HeavyChart'` and has no choice but to include it in the bundle. It cannot know that the user won't visit the `/orders` page immediately.

5.  **What we need**: A way to tell Next.js: "Don't bundle this component initially. Only load its code when the user actually navigates to the page that needs it." This is called **code splitting** on a route level, and the technique is **lazy loading**.

### Iteration 1: Slaying the Monolith with `next/dynamic`

To solve this, we'll use `next/dynamic`, a special function that allows for dynamic, client-side loading of React components. It tells Next.js to create a separate JavaScript chunk for the specified component.

**Before** (Iteration 0): The `OrdersPage` directly imports `HeavyChart`.

```tsx
// src/app/orders/page.tsx - Version 0
import HeavyChart from "@/components/HeavyChart";

export default function OrdersPage() {
  return (
    <div>
      <h1 className="text-3xl font-bold mb-6">Orders</h1>
      <HeavyChart />
    </div>
  );
}
```

**After** (Iteration 1): We use `next/dynamic` to load `HeavyChart` lazily.

```tsx
// src/app/orders/page.tsx - Version 1
import dynamic from "next/dynamic";

// The dynamic import creates a separate chunk for HeavyChart.
const HeavyChart = dynamic(() => import("@/components/HeavyChart"), {
  // Optional: Show a loading component while the dynamic component is loading.
  loading: () => <p>Loading chart...</p>,
  // Optional: Disable Server-Side Rendering for this component if it relies on browser APIs.
  ssr: false, 
});

export default function OrdersPage() {
  return (
    <div>
      <h1 className="text-3xl font-bold mb-6">Orders</h1>
      <HeavyChart />
    </div>
  );
}
```

### Verification: Checking the Results

Let's clear the cache and reload the homepage (`/`) again, observing the Network tab.

**Browser DevTools - Network Tab** (on initial load of `/`):

```
Name                    Status  Type      Size
----------------------------------------------------
page-....js             200     script    25 KB   <-- IMPROVEMENT!
...other chunks
```

The initial JavaScript chunk is now tiny! The code for `HeavyChart` is gone.

Now, navigate to the `/orders` page.

**Browser DevTools - Network Tab** (when navigating to `/orders`):

```
Name                    Status  Type      Size
----------------------------------------------------
components_HeavyChart_tsx-....js  200  script  500 KB  <-- Loaded on demand
```

A new chunk containing `HeavyChart` is downloaded only when we navigate to the page that uses it.

**Performance Metrics** (Simulated Slow 3G connection):
- **Before**:
  - Bundle size: 525 KB
  - LCP: 3.2s
- **After**:
  - Bundle size: 25 KB (95% reduction)
  - LCP: 0.8s (75% improvement)

**Improvement**: The initial page load is dramatically faster. Users who never visit the "Orders" page will never download the 500KB charting library, saving bandwidth and improving their experience.

### When to Apply This Solution

-   **What it optimizes for**: Fast initial page loads (Time to Interactive, First Contentful Paint).
-   **What it sacrifices**: A small delay on the first navigation to the lazy-loaded route while the new chunk is downloaded.
-   **When to choose this approach**:
    -   For pages or large components that are not visible on the initial load.
    -   For features behind a login wall.
    -   For components with heavy dependencies (e.g., charting, mapping, or 3D rendering libraries).
-   **When to avoid this approach**:
    -   For small components where the overhead of a separate network request outweighs the bundle size savings.
    -   For components critical to the initial view (e.g., the navigation bar itself).

**Limitation preview**: While we've solved the initial bundle size, our data loading is still naive. In the next section, we'll see how fetching data on the client side after the page loads creates a new set of performance problems.

## Route-based data loading

## The Client-Side Fetching Waterfall

Our application now has excellent code splitting. The initial load is fast, and secondary pages are loaded on demand. But what about the *data* for those pages? A common pattern, especially for developers coming from Single-Page Applications (SPAs), is to fetch data inside a `useEffect` hook.

Let's refactor our `ProductsPage` to fetch its data from an API endpoint.

### Iteration 1 Recap

Our `ProductsPage` currently uses a hardcoded array.

```tsx
// src/app/products/page.tsx - Version 0
export default function ProductsPage() {
  const products = Array.from({ length: 100 }, (_, i) => `Product ${i + 1}`);
  // ... renders products
}
```

### Iteration 2: Introducing Client-Side Data Fetching

First, let's create a simple API route to serve the product data.

<code language="typescript">
// src/app/api/products/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  // Simulate a database delay
  await new Promise(resolve => setTimeout(resolve, 500)); 

  const products = Array.from({ length: 100 }, (_, i) => ({
    id: i + 1,
    name: `Product ${i + 1}`,
  }));

  return NextResponse.json(products);
}
```

Now, let's modify the `ProductsPage` to fetch from this endpoint using the classic `useEffect` pattern. This requires converting it to a Client Component.

```tsx
// src/app/products/page.tsx - Version 1 (Problematic)
"use client";

import { useState, useEffect } from "react";

interface Product {
  id: number;
  name: string;
}

export default function ProductsPage() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setIsLoading(false);
      });
  }, []);

  if (isLoading) {
    return <p>Loading products...</p>;
  }

  return (
    <div>
      <h1 className="text-3xl font-bold mb-6">Products</h1>
      <ul className="space-y-2">
        {products.map((product) => (
          <li key={product.id} className="p-2 border rounded">
            {product.name}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

This code works, but it introduces a subtle performance issue known as a "request waterfall." Let's diagnose it.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
When the user navigates to `/products`, they first see "Loading products...". After a noticeable delay, the loading text is replaced by the actual product list. This flash of loading content can be jarring and contributes to Content Layout Shift (CLS).

**Browser DevTools - Network Tab** (Navigate from `/` to `/products`):

You will see a sequence like this:

1.  **Request 1**: `products-page-....js` (The JavaScript for the page component)
2.  **Request 2**: `products` (The XHR/fetch request to `/api/products`)

This is a waterfall: the browser can't even *start* fetching the data until *after* it has downloaded, parsed, and executed the JavaScript for the `ProductsPage` component.

```
Request Timeline:
|--------------------|  products-page-....js (200ms)
                     |----------------|  /api/products (500ms)
                     ^                ^
                     |                Content appears here
                     Page navigation starts
```

**Performance Metrics**:
- **Time to Contentful Data**: ~700ms (200ms for JS + 500ms for API) + rendering time.
- **Content Layout Shift (CLS)**: A non-zero value, as the loading indicator is replaced by the list.

**Let's parse this evidence**:

1.  **What the user experiences**: A loading spinner followed by the content popping in. The page feels slow to load its essential data.

    -   **Expected**: The page should load with its data already present, or at least start loading the data and the page code in parallel.
    -   **Actual**: A sequential load: Page Code -> Data Fetch -> Render.

2.  **What the Network tab reveals**: A clear waterfall pattern. The data fetch depends on the component code fetch.

    -   **Key indicator**: The `fetch('/api/products')` request starts *after* the component's JavaScript chunk finishes downloading.

3.  **Root cause identified**: Client-side data fetching (`useEffect`) can only run after the component has mounted in the browser. This inherently creates a waterfall, delaying the presentation of meaningful content to the user.

4.  **Why the current approach can't solve this**: The `useEffect` hook is fundamentally a client-side browser mechanism. It cannot run on the server. This forces the data fetching to happen late in the rendering lifecycle.

5.  **What we need**: A way to fetch data *on the server* before the page is even sent to the client. The server should handle the data fetching, render the complete HTML with the data included, and stream it to the browser.

### Iteration 3: Server Components for Route-Based Data Loading

The App Router in Next.js solves this problem elegantly with React Server Components (RSCs). By making our page component `async`, we can fetch data directly within it. This code runs exclusively on the server.

**Before** (Iteration 2): A Client Component with `useEffect`.

```tsx
// src/app/products/page.tsx - Version 1 (Problematic)
"use client";

import { useState, useEffect } from "react";
// ... (rest of the client-side code)
```

**After** (Iteration 3): A Server Component that fetches data directly.

```tsx
// src/app/products/page.tsx - Version 2 (Optimized)

interface Product {
  id: number;
  name: string;
}

async function getProducts(): Promise<Product[]> {
  // In a real app, you'd fetch from your database or an external API.
  // We fetch from our own route handler for this example.
  // The `cache: 'no-store'` option ensures fresh data on every request.
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/products`, {
    cache: 'no-store' 
  });
  
  if (!res.ok) {
    throw new Error('Failed to fetch products');
  }

  return res.json();
}

// The page component is now an async function!
export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div>
      <h1 className="text-3xl font-bold mb-6">Products</h1>
      <ul className="space-y-2">
        {products.map((product) => (
          <li key={product.id} className="p-2 border rounded">
            {product.name}
          </li>
        ))}
      </ul>
    </div>
  );
}
```
*Note: You'll need to set `NEXT_PUBLIC_API_URL=http://localhost:3000` in a `.env.local` file for this to work.*

### Verification: A Waterfall-Free World

Let's navigate to the `/products` page again and inspect the Network tab.

**Browser Behavior**:
The page loads, and the product list is immediately visible. There is no "Loading products..." flash. The transition feels much smoother.

**Browser DevTools - Network Tab**:
When you navigate to `/products`, you will see the request for the page's HTML. You will **not** see a separate client-side `fetch` request to `/api/products`. The data is already embedded in the HTML streamed from the server.

**Performance Metrics**:
- **Time to Contentful Data**: The time it takes for the server to respond with the initial HTML chunk, which already contains the data. This is much faster from a user's perspective.
- **Content Layout Shift (CLS)**: 0. The page renders in its final state, eliminating layout shifts from loading indicators.

**Improvement**: We have eliminated the client-side request waterfall. The server does the work, leading to a faster perceived load time, better SEO, and a superior user experience.

**Limitation preview**: Our app is now fast and efficient. But what about the user's context? When they scroll down a long list, navigate away, and come back, they lose their place. In the next section, we'll tackle scroll restoration and focus management to perfect the navigation experience.

## Scroll restoration and focus management

## The Amnesiac App: Forgetting the User's Context

Our dashboard is now performant, with code-split pages and server-rendered data. But a great user experience is about more than just speed; it's about feeling intuitive and seamless. One common UX failure is losing the user's scroll position after navigation.

### Iteration 2 Recap

Our `ProductsPage` renders a list of 100 items. It's fast and data is loaded on the server.

```tsx
// src/app/products/page.tsx - Version 2
export default async function ProductsPage() {
  const products = await getProducts();
  // ... renders a long list of products
}
```

### The Problem: Lost Scroll Position

Let's demonstrate the problem. To do this, we need a page to navigate *to*.

<code language="tsx">
// src/app/products/[id]/page.tsx
export default function ProductDetailPage({ params }: { params: { id: string } }) {
  return (
    <div>
      <h1 className="text-3xl font-bold">Product Detail: {params.id}</h1>
      <p>Details about product {params.id} would go here.</p>
    </div>
  );
}
```

And let's update our product list to link to this new detail page.

<code language="tsx">
// src/app/products/page.tsx - Version 2.1 (for demonstration)
import Link from 'next/link';

// ... (getProducts function is the same)

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div>
      <h1 className="text-3xl font-bold mb-6">Products</h1>
      <ul className="space-y-2">
        {products.map((product) => (
          <li key={product.id} className="p-2 border rounded">
            <Link href={`/products/${product.id}`} className="block">
              {product.name}
            </Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

**Failure Scenario**:
1.  Navigate to the `/products` page.
2.  Scroll down to "Product 75".
3.  Click on the link for "Product 75". You are now on `/products/75`.
4.  Click the browser's "Back" button.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
When you return to the `/products` page, you are at the **top of the page**, not back at "Product 75". You have lost your context. This is incredibly frustrating for users navigating long lists or feeds.

**Let's parse this evidence**:

1.  **What the user experiences**: Disorientation and frustration. They have to perform extra work (scrolling) to get back to where they were.

    -   **Expected**: The browser should remember my scroll position and restore it when I navigate back.
    -   **Actual**: The page loads at the top, as if visited for the first time.

2.  **Root cause identified**: By default, the Next.js App Router *does* attempt to restore scroll position. However, this behavior can sometimes be misconfigured or disabled. The key is to understand how it works and how to ensure it functions correctly.

### Solution: Understanding and Ensuring Scroll Restoration

In the Next.js App Router, scroll restoration is **enabled by default**. The behavior we just described is what you'd expect in older frameworks or if the feature were disabled. The App Router's router maintains a history of scroll positions and automatically restores them on back/forward navigation.

So, why might it fail?
*   **Custom Layouts**: Complex layouts with internal scroll containers (`overflow: scroll`) might not be tracked automatically. The router tracks the `window` scroll position.
*   **Disabling the Feature**: You can explicitly disable it on a `<Link>` component with `scroll={false}`. This is useful for links that only update a small part of the page (like a tab) where you *don't* want the page to scroll to the top.

Let's demonstrate the `scroll={false}` prop. If we wanted a link to *not* scroll to the top on navigation, we would do this:

<code language="tsx">
// Example of disabling scroll behavior
<Link href="/some-other-page" scroll={false}>
  This link won't scroll the page to the top.
</Link>
```

Since the default behavior is what we want, there's no code to "fix". The important lesson is that Next.js handles this for us, and we should be careful not to break it.

### The Next Frontier: Accessibility and Focus Management

While scroll position is handled, a related and often overlooked issue is **focus management**. After navigating to a new page, where is the browser's focus?

**Failure Scenario**:
1.  Use your keyboard's `Tab` key to navigate through the links on the `/` page.
2.  Press `Enter` on the "Products" link.
3.  The `/products` page loads.
4.  Now, press the `Tab` key again. Where does the focus go?

**Browser Behavior**:
The focus is often lost or unpredictable. It might jump to the browser's URL bar or remain "in limbo". For users relying on screen readers or keyboard navigation, this is a major accessibility failure. The context is lost.

**What we need**: A system that, upon every route change, programmatically moves the browser's focus to a logical starting point on the new page, typically the main heading (`<h1>`).

### Iteration 4: Implementing a Focus Manager

We can solve this with a small, reusable client component that listens for route changes and manages focus.

**Project Structure**:
```
src/
└── components/
    ├── RouteFocusManager.tsx  # New component
    └── ...
```

<code language="tsx">
// src/components/RouteFocusManager.tsx
"use client";

import { usePathname } from "next/navigation";
import { useEffect, useRef } from "react";

export function RouteFocusManager() {
  const pathname = usePathname();
  const mainContentRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    // Find the main content area. We'll add an id="main-content" to it.
    mainContentRef.current = document.getElementById("main-content");

    // On route change, focus the main content area.
    if (mainContentRef.current) {
      mainContentRef.current.focus();
    }
  }, [pathname]);

  return null; // This component renders nothing.
}
```

Now, we need to integrate this into our root layout and add the corresponding `id` and `tabIndex` to our main content element.

<code language="tsx">
// src/app/layout.tsx - Version 2
import Nav from "@/components/Nav";
import { RouteFocusManager } from "@/components/RouteFocusManager"; // Import
import "./globals.css";

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        {/* This component is a client component that will run in the browser */}
        <RouteFocusManager /> 
        <Nav />
        {/* Add id and tabIndex to the main element */}
        <main id="main-content" tabIndex={-1} className="p-8 outline-none">
          {children}
        </main>
      </body>
    </html>
  );
}
```

**Why `tabIndex={-1}`?** This makes an element programmatically focusable (via `element.focus()`) without adding it to the natural tab order. This is exactly what we want.

### Verification: Accessible Navigation

1.  Navigate the site using your keyboard.
2.  Click `Enter` on a link in the navigation.
3.  As soon as the new page loads, press `Tab`.

**Browser Behavior**:
The focus now correctly starts from the top of the main content area. The first press of `Tab` will focus the first focusable element *within* the new page's content (e.g., the first product link), not the browser chrome or some other random element. Screen readers will announce the new page's main heading, giving users immediate context.

**Improvement**: Our application is now significantly more accessible. We provide a predictable and seamless experience for all users, including those who rely on assistive technologies.

**Limitation preview**: Our app feels great to use. But can we make it feel *instantaneous*? The final piece of the puzzle is to load the code for the next page *before* the user even clicks the link. This is called prefetching.

## When to prefetch

## Predicting the Future: Making Navigation Instantaneous

We've optimized our bundle size, data loading, and navigation context. The final step in mastering routing is to make navigation feel truly instantaneous. Even with code splitting, there's a small delay when a user clicks a link while the browser fetches the necessary JavaScript for the new page. Prefetching solves this by loading resources ahead of time.

### Iteration 3 Recap

Our application is well-structured. When a user clicks the "Orders" link, the browser requests the JavaScript chunk for the orders page and then renders it.

### The Problem: Network Latency on Navigation

Let's simulate a user on a slightly slower network to see the problem clearly.

**Failure Scenario**:
1.  Open Browser DevTools and go to the Network tab.
2.  Select a "Fast 3G" or similar network throttling profile.
3.  Load the homepage (`/`).
4.  Click the "Orders" link.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
There is a noticeable pause—perhaps 1-2 seconds—between clicking the "Orders" link and seeing the "Loading chart..." message appear. The UI feels sluggish.

**Browser DevTools - Network Tab**:
You can see the exact cause of the delay. The request for the `orders-page-....js` chunk only begins *after* the user clicks the link. The navigation is blocked on this network request.

```
User Action: Click "Orders" Link
|
|--- Network Request for orders-page-....js (e.g., 800ms on Fast 3G) ---|
                                                                         |
                                                                         Page Renders
```

**Let's parse this evidence**:

1.  **What the user experiences**: A laggy interface. The app doesn't feel responsive.

    -   **Expected**: Clicking a link should feel like an instant transition.
    -   **Actual**: The transition is gated by the time it takes to download the next page's code.

2.  **What the Network tab reveals**: The code for the destination route is fetched on-demand, triggered by the click event.

    -   **Key indicator**: The waterfall shows the JS chunk download starting at the moment of the click.

3.  **Root cause identified**: We are loading the code *reactively* (in response to a click) instead of *proactively* (in anticipation of a click).

4.  **What we need**: A way to tell Next.js, "I think the user might click this link soon, so please download the code for it in the background *before* they click."

### Solution: Automatic Prefetching with `<Link>`

This is another area where Next.js provides a powerful, built-in solution. The `<Link>` component automatically handles this for us.

**By default, `<Link>` prefetches the code for the linked route when the link enters the user's viewport.**

This means that for our `Nav` component, as soon as it's visible on the screen, Next.js will start downloading the JavaScript chunks for the `/orders` and `/products` pages in the background with a low priority.

### Verification: Prefetching in Action

1.  Disable network throttling.
2.  Go to the Network tab in DevTools and clear it.
3.  Reload the homepage (`/`).

**Browser DevTools - Network Tab**:
Almost immediately after the page loads, you will see new, low-priority requests for the JavaScript associated with the other pages.

```
Name                         Status  Type      Initiator
----------------------------------------------------------
... initial page chunks ...
orders-page-....js           200     script    Link (prefetch)
products-page-....js         200     script    Link (prefetch)
```

The "Initiator" column often indicates that the prefetch was triggered by the `<Link>` component. Now, when you click the "Orders" or "Products" link, the browser doesn't need to fetch the code from the network; it's already in the browser's cache. The navigation is instant.

### When to Prefetch, and When Not To

Automatic prefetching is a fantastic default, but it's not always the right choice. You need to make conscious decisions based on the user experience and performance trade-offs.

#### When to use default prefetching (the happy path)
This is ideal for primary navigation and any high-likelihood next steps. The "Orders" and "Products" links in our nav bar are perfect candidates. The cost of a small background download is well worth the benefit of instant navigation.

#### When to disable prefetching
You can disable this behavior with the `prefetch={false}` prop.

<code language="tsx">
<Link href="/rarely-visited-page" prefetch={false}>
  Annual Report Generator
</Link>
```

**Use cases for disabling prefetching**:
-   **Very Large, Rarely-Visited Pages**: If our "Orders" page included a 5MB mapping library and was only visited by 1% of users, prefetching it for everyone would be a waste of bandwidth.
-   **Pages with Frequently Changing Data**: Prefetching only fetches the *code*, not the *data*. But in some edge cases with rapidly changing UIs, you might want to avoid it.
-   **UI with Hundreds of Links**: Imagine a dashboard with a table of 500 orders, each linking to its detail page. Prefetching all 500 would clog the user's network connection. In this case, Next.js is smart enough to not prefetch them all, but it's a scenario to be mindful of.

#### When to prefetch programmatically
Sometimes, the best time to prefetch isn't when a link is in the viewport, but in response to another user action. For this, we can use the `router.prefetch()` method from the `useRouter` hook.

**Use Case**: Imagine a "Create Product" wizard with multiple steps. When the user successfully completes Step 1, we have very high confidence they will proceed to Step 2. This is the perfect moment to programmatically prefetch the code for Step 2.

<code language="tsx">
// src/components/Step1Form.tsx
"use client";

import { useRouter } from "next/navigation";

export default function Step1Form() {
  const router = useRouter();

  // Prefetch Step 2 when the user starts interacting with the form
  const handleFocus = () => {
    router.prefetch('/products/create/step-2');
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    // ... submit form data
    router.push('/products/create/step-2');
  };

  return (
    <form onSubmit={handleSubmit} onFocus={handleFocus}>
      {/* Form fields for step 1 */}
      <button type="submit">Next Step</button>
    </form>
  );
}
```

### Decision Framework: Which Prefetching Strategy to Use?

| Scenario                               | Default `<Link>` (in viewport) | `<Link prefetch={false}>` | `router.prefetch()` (programmatic) |
| -------------------------------------- | ------------------------------ | ------------------------- | ---------------------------------- |
| **Main site navigation**               | ✅ **Best Choice**             | ❌ Slower navigation      | Overkill                           |
| **Link to a heavy, optional tool**     | ❌ Wastes bandwidth            | ✅ **Best Choice**        | Unnecessary                        |
| **Table with 100s of links to items**  | ⚠️ Okay, but be aware          | ✅ **Safe Choice**        | Too complex                        |
| **High-confidence next action (wizard)** | ✅ Works fine                  | ❌ Slower navigation      | ✅ **Optimal Choice**              |
| **Link in a page footer**              | ✅ Fine, low priority          | ❌ Unnecessary optimization | Overkill                           |

By understanding and applying these prefetching strategies, you gain fine-grained control over your application's network behavior, ensuring a user experience that feels incredibly fast and responsive.

## Synthesis - The Complete Journey

## The Journey: From Problem to Solution

We have taken our simple E-commerce Admin Dashboard through a series of transformations, with each step addressing a specific performance or user experience failure. This journey reflects the real-world process of building and optimizing a modern web application.

| Iteration | Failure Mode                               | Technique Applied                        | Result                                        | Performance Impact                               |
| :-------- | :----------------------------------------- | :--------------------------------------- | :-------------------------------------------- | :----------------------------------------------- |
| **0**     | **Monolithic Bundle**                      | None (Naive Implementation)              | Slow initial load, all code sent at once.     | High LCP, large initial JS bundle (~525KB).      |
| **1**     | **Slow Initial Load**                      | Code Splitting with `next/dynamic`       | Drastically smaller initial bundle.           | 95% reduction in initial bundle size, faster LCP. |
| **2**     | **Client-Side Data Waterfall**             | Server Component Data Fetching (`async`) | Data is fetched on the server, no spinners.   | Eliminated client-side waterfall, CLS is 0.      |
| **3**     | **Lost User Context** (Scroll & Focus)     | Focus Management Component               | Scroll position restored, focus is managed.   | Improved accessibility (a11y) and UX.            |
| **4**     | **Laggy Navigation**                       | `<Link>` Prefetching (default)           | Code for next pages is loaded in background.  | Perceived navigation becomes instantaneous.      |

### Final Implementation

Here is the final, optimized code for our `products` page, incorporating the lessons learned. It's a Server Component that fetches its own data, and its route is automatically code-split and prefetched by the `Nav` component.

<code language="tsx">
// src/app/products/page.tsx (Final Version)
import Link from 'next/link';

interface Product {
  id: number;
  name: string;
}

async function getProducts(): Promise<Product[]> {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/products`, {
    cache: 'no-store' 
  });
  
  if (!res.ok) {
    throw new Error('Failed to fetch products');
  }

  return res.json();
}

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div>
      <h1 className="text-3xl font-bold mb-6">Products</h1>
      <ul className="space-y-2">
        {products.map((product) => (
          <li key={product.id} className="p-2 border rounded">
            {/* 
              - This link will be prefetched if it enters the viewport.
              - Navigation will not cause a page refresh.
              - Scroll position will be restored on back/forward navigation.
            */}
            <Link href={`/products/${product.id}`} className="block">
              {product.name}
            </Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

And our root layout ensures an accessible experience for all users.

<code language="tsx">
// src/app/layout.tsx (Final Version)
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import Nav from "@/components/Nav";
import { RouteFocusManager } from "@/components/RouteFocusManager";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "Admin Dashboard",
  description: "E-commerce Admin",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <RouteFocusManager />
        <Nav />
        <main id="main-content" tabIndex={-1} className="p-8 outline-none">
          {children}
        </main>
      </body>
    </html>
  );
}
```

### Lessons Learned

1.  **Code-Split by Default**: Structure your application around routes. Next.js's file-system based router automatically code-splits your pages. Use `next/dynamic` for large components *within* a page that are conditionally rendered.
2.  **Fetch Data on the Server**: Embrace Server Components. Co-locating your data fetching with your components eliminates client-side waterfalls, improves performance, and simplifies your code. Avoid `useEffect` for data fetching unless you specifically need client-side interactivity with that data.
3.  **Trust the Platform, but Verify**: Next.js provides excellent defaults for routing, like scroll restoration and prefetching. Understand how they work so you can leverage them effectively and know when to opt-out for specific use cases.
4.  **Accessibility is Not an Afterthought**: A performant app that is unusable by a portion of your audience is a failed app. Implement focus management from the start to ensure a robust experience for keyboard and screen reader users.
5.  **Think Proactively, Not Reactively**: Prefetching embodies this principle. By anticipating the user's next move, you can create an experience that feels faster than it has any right to be.

By mastering these advanced routing patterns, you move beyond simply building applications that *work* to crafting experiences that are fast, resilient, and a joy to use.
