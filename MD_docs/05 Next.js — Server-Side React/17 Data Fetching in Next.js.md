# Chapter 17: Data Fetching in Next.js

## Server Components: fetch directly

## The Problem: Client-Side Data Fetching Waterfalls

Let's continue with our **E-commerce Product Catalog** from Chapter 16. We have a product listing page that needs to display products, categories, and featured items. In a traditional React app, we'd fetch all this data on the client.

Here's what that looks like—and why it's problematic.

**Project Structure**:
```
src/
├── app/
│   ├── products/
│   │   └── page.tsx          ← Product listing page
│   └── layout.tsx
├── components/
│   ├── ProductGrid.tsx       ← Displays products
│   ├── CategoryFilter.tsx    ← Category sidebar
│   └── FeaturedBanner.tsx    ← Featured products
└── lib/
    └── api.ts                ← API client functions
```

### Iteration 0: The Client-Side Waterfall (The Failure)

Let's build this the "React way"—fetching everything on the client with `useEffect`.

```typescript
// src/lib/api.ts
export interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  imageUrl: string;
  featured: boolean;
}

export interface Category {
  id: string;
  name: string;
  count: number;
}

// Simulated API calls (in production, these would be real endpoints)
export async function getProducts(): Promise<Product[]> {
  await new Promise(resolve => setTimeout(resolve, 800)); // Simulate network delay
  return [
    { id: '1', name: 'Laptop Pro', price: 1299, category: 'electronics', imageUrl: '/laptop.jpg', featured: true },
    { id: '2', name: 'Wireless Mouse', price: 29, category: 'electronics', imageUrl: '/mouse.jpg', featured: false },
    { id: '3', name: 'Desk Chair', price: 399, category: 'furniture', imageUrl: '/chair.jpg', featured: false },
    // ... more products
  ];
}

export async function getCategories(): Promise<Category[]> {
  await new Promise(resolve => setTimeout(resolve, 600));
  return [
    { id: 'electronics', name: 'Electronics', count: 45 },
    { id: 'furniture', name: 'Furniture', count: 23 },
    { id: 'clothing', name: 'Clothing', count: 67 },
  ];
}

export async function getFeaturedProducts(): Promise<Product[]> {
  await new Promise(resolve => setTimeout(resolve, 700));
  const products = await getProducts();
  return products.filter(p => p.featured);
}
```

```tsx
// src/app/products/page.tsx - Client-side approach (PROBLEMATIC)
'use client';

import { useEffect, useState } from 'react';
import { getProducts, getCategories, getFeaturedProducts, Product, Category } from '@/lib/api';

export default function ProductsPage() {
  const [products, setProducts] = useState<Product[]>([]);
  const [categories, setCategories] = useState<Category[]>([]);
  const [featured, setFeatured] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    async function fetchData() {
      setIsLoading(true);
      
      // Sequential fetching - each waits for the previous
      const productsData = await getProducts();
      setProducts(productsData);
      
      const categoriesData = await getCategories();
      setCategories(categoriesData);
      
      const featuredData = await getFeaturedProducts();
      setFeatured(featuredData);
      
      setIsLoading(false);
    }
    
    fetchData();
  }, []);

  if (isLoading) {
    return <div>Loading products...</div>;
  }

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      
      <div className="grid grid-cols-12 gap-8">
        {/* Category sidebar */}
        <aside className="col-span-3">
          <h2 className="text-xl font-semibold mb-4">Categories</h2>
          <ul className="space-y-2">
            {categories.map(cat => (
              <li key={cat.id}>
                {cat.name} ({cat.count})
              </li>
            ))}
          </ul>
        </aside>

        {/* Main content */}
        <main className="col-span-9">
          {/* Featured banner */}
          {featured.length > 0 && (
            <div className="mb-8 p-6 bg-blue-50 rounded-lg">
              <h2 className="text-2xl font-semibold mb-4">Featured Products</h2>
              <div className="grid grid-cols-3 gap-4">
                {featured.map(product => (
                  <div key={product.id} className="text-center">
                    <img src={product.imageUrl} alt={product.name} className="w-full h-32 object-cover rounded" />
                    <p className="mt-2 font-medium">{product.name}</p>
                    <p className="text-blue-600">${product.price}</p>
                  </div>
                ))}
              </div>
            </div>
          )}

          {/* Product grid */}
          <div className="grid grid-cols-3 gap-6">
            {products.map(product => (
              <div key={product.id} className="border rounded-lg p-4">
                <img src={product.imageUrl} alt={product.name} className="w-full h-48 object-cover rounded" />
                <h3 className="mt-4 font-semibold">{product.name}</h3>
                <p className="text-gray-600">{product.category}</p>
                <p className="mt-2 text-lg font-bold">${product.price}</p>
              </div>
            ))}
          </div>
        </main>
      </div>
    </div>
  );
}
```

### Diagnostic Analysis: Reading the Waterfall Failure

Let's run this in the browser and observe what happens.

**Browser Behavior**:
- User navigates to `/products`
- Sees blank page with "Loading products..." for 2+ seconds
- Then entire page appears at once
- No progressive loading—everything or nothing

**Browser Console Output**:
```
[No errors, but let's add timing logs]
```

Let's add console.log statements to see the timing:

```tsx
// Modified useEffect with timing logs
useEffect(() => {
  async function fetchData() {
    console.log('[0ms] Starting data fetch...');
    setIsLoading(true);
    
    const start1 = performance.now();
    const productsData = await getProducts();
    console.log(`[${Math.round(performance.now())}ms] Products loaded (took ${Math.round(performance.now() - start1)}ms)`);
    setProducts(productsData);
    
    const start2 = performance.now();
    const categoriesData = await getCategories();
    console.log(`[${Math.round(performance.now())}ms] Categories loaded (took ${Math.round(performance.now() - start2)}ms)`);
    setCategories(categoriesData);
    
    const start3 = performance.now();
    const featuredData = await getFeaturedProducts();
    console.log(`[${Math.round(performance.now())}ms] Featured loaded (took ${Math.round(performance.now() - start3)}ms)`);
    setFeatured(featuredData);
    
    setIsLoading(false);
    console.log(`[${Math.round(performance.now())}ms] All data loaded, rendering page`);
  }
  
  fetchData();
}, []);
```

**Browser Console Output** (actual timing):
```
[0ms] Starting data fetch...
[823ms] Products loaded (took 823ms)
[1456ms] Categories loaded (took 633ms)
[2189ms] Featured loaded (took 733ms)
[2189ms] All data loaded, rendering page
```

**Network Tab Analysis**:
- Filter: Fetch/XHR
- Observation: Three requests fire **sequentially**, not in parallel
- Timeline:
  - 0-800ms: `/api/products` (waiting)
  - 800-1400ms: `/api/categories` (waiting)
  - 1400-2100ms: `/api/featured` (waiting)
- Pattern: Classic waterfall—each request waits for the previous to complete
- Total time: 2.2 seconds
- Wasted time: ~1.4 seconds (requests could have run in parallel)

**React DevTools Evidence**:
- `ProductsPage` component selected
- State: `{ products: [], categories: [], featured: [], isLoading: true }`
- After 2.2 seconds: State updates three times sequentially
- Render count: 4 renders (initial + 3 state updates)

**Performance Metrics**:
- **Time to First Byte (TTFB)**: 50ms (HTML arrives quickly)
- **First Contentful Paint (FCP)**: 100ms (shows "Loading products...")
- **Largest Contentful Paint (LCP)**: 2300ms ⚠️ (waits for all data)
- **Time to Interactive (TTI)**: 2400ms ⚠️
- **Total Blocking Time (TBT)**: 150ms (React hydration + rendering)

### Let's Parse This Evidence

1. **What the user experiences**:
   - Expected: See the page structure immediately, with content loading progressively
   - Actual: Stare at a loading spinner for 2+ seconds, then everything appears at once

2. **What the console reveals**:
   - Key indicator: Sequential timing—each fetch waits for the previous
   - Error location: Not an error, but a design flaw in the data fetching strategy

3. **What the Network tab shows**:
   - Technical evidence: Waterfall pattern—requests are serialized
   - Root cause: `await` statements create sequential dependencies
   - Wasted opportunity: These three requests are independent and could run in parallel

4. **Root cause identified**: 
   Sequential async/await creates an artificial dependency chain. The categories fetch doesn't need to wait for products, and featured doesn't need to wait for categories.

5. **Why the current approach can't solve this**:
   Even if we parallelize the fetches with `Promise.all()`, we still have fundamental problems:
   - All data fetching happens **after** the JavaScript bundle loads and executes
   - The server already has access to the database—why send it to the client first?
   - The HTML sent to the browser is empty—terrible for SEO and perceived performance
   - Users on slow connections wait even longer

6. **What we need**:
   A way to fetch data **on the server** before sending HTML to the client, so users see content immediately instead of spinners.

### The Fundamental Problem: Client-Side Fetching in a Server-Capable Framework

This approach has multiple issues:

1. **Waterfall by default**: Sequential fetches waste time
2. **Empty HTML**: View source shows no content—bad for SEO
3. **JavaScript required**: Page is blank until JS loads and executes
4. **Wasted server capability**: Next.js can fetch on the server, but we're not using it
5. **Poor perceived performance**: Users see loading states instead of content

**What we need**: Fetch data on the server, render HTML with content, send that to the client. This is what Server Components enable.

## Server Components: Fetching Where the Data Lives

Next.js App Router introduces **Server Components**—components that run only on the server, never in the browser. They can fetch data directly, access databases, read environment variables, and render HTML that's sent to the client.

### The Mental Model Shift

**Client Component** (traditional React):
```
Browser → Load JS → Execute component → Fetch data → Render → Show content
         [Empty HTML]  [Spinner shown]   [Network]   [Finally!]
```

**Server Component** (Next.js App Router):
```
Server → Fetch data → Render component → Send HTML → Browser displays
        [Fast!]       [On server]        [Content!]   [Immediately!]
```

### Key Characteristics of Server Components

1. **Run on the server**: Code executes during the build (static) or on each request (dynamic)
2. **Can fetch directly**: No need for API routes—just call your database or external APIs
3. **Zero JavaScript to client**: The component code never ships to the browser
4. **Can use server-only code**: Access environment variables, file system, databases directly
5. **Cannot use hooks**: No `useState`, `useEffect`, or event handlers (those require client interactivity)

### Iteration 1: Server Component Data Fetching

Let's refactor our product page to use Server Components. By default, all components in the App Router are Server Components unless marked with `'use client'`.

```tsx
// src/app/products/page.tsx - Server Component approach
import { getProducts, getCategories, getFeaturedProducts } from '@/lib/api';

// This is a Server Component by default (no 'use client' directive)
export default async function ProductsPage() {
  // Fetch data directly in the component - this runs on the server
  const products = await getProducts();
  const categories = await getCategories();
  const featured = await getFeaturedProducts();

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      
      <div className="grid grid-cols-12 gap-8">
        {/* Category sidebar */}
        <aside className="col-span-3">
          <h2 className="text-xl font-semibold mb-4">Categories</h2>
          <ul className="space-y-2">
            {categories.map(cat => (
              <li key={cat.id}>
                {cat.name} ({cat.count})
              </li>
            ))}
          </ul>
        </aside>

        {/* Main content */}
        <main className="col-span-9">
          {/* Featured banner */}
          {featured.length > 0 && (
            <div className="mb-8 p-6 bg-blue-50 rounded-lg">
              <h2 className="text-2xl font-semibold mb-4">Featured Products</h2>
              <div className="grid grid-cols-3 gap-4">
                {featured.map(product => (
                  <div key={product.id} className="text-center">
                    <img src={product.imageUrl} alt={product.name} className="w-full h-32 object-cover rounded" />
                    <p className="mt-2 font-medium">{product.name}</p>
                    <p className="text-blue-600">${product.price}</p>
                  </div>
                ))}
              </div>
            </div>
          )}

          {/* Product grid */}
          <div className="grid grid-cols-3 gap-6">
            {products.map(product => (
              <div key={product.id} className="border rounded-lg p-4">
                <img src={product.imageUrl} alt={product.name} className="w-full h-48 object-cover rounded" />
                <h3 className="mt-4 font-semibold">{product.name}</h3>
                <p className="text-gray-600">{product.category}</p>
                <p className="mt-2 text-lg font-bold">${product.price}</p>
              </div>
            ))}
          </div>
        </main>
      </div>
    </div>
  );
}
```

### What Changed?

**Before** (Client Component):
```tsx
'use client';
const [products, setProducts] = useState<Product[]>([]);
useEffect(() => { /* fetch */ }, []);
```

**After** (Server Component):
```tsx
// No 'use client' directive
async function ProductsPage() {  // ← Component is async
  const products = await getProducts();  // ← Direct fetch
  // No useState, no useEffect, no loading state
}
```

### Verification: Does This Actually Work?

Let's run this and observe the difference.

**Browser Behavior**:
- User navigates to `/products`
- Page appears **immediately** with all content visible
- No loading spinner—content is already in the HTML

**View Source** (Right-click → View Page Source):
```html
<!DOCTYPE html>
<html>
<body>
  <div class="container mx-auto px-4 py-8">
    <h1 class="text-3xl font-bold mb-8">Our Products</h1>
    <div class="grid grid-cols-12 gap-8">
      <aside class="col-span-3">
        <h2 class="text-xl font-semibold mb-4">Categories</h2>
        <ul class="space-y-2">
          <li>Electronics (45)</li>
          <li>Furniture (23)</li>
          <li>Clothing (67)</li>
        </ul>
      </aside>
      <main class="col-span-9">
        <!-- Full product HTML is here! -->
        <div class="grid grid-cols-3 gap-6">
          <div class="border rounded-lg p-4">
            <img src="/laptop.jpg" alt="Laptop Pro">
            <h3 class="mt-4 font-semibold">Laptop Pro</h3>
            <p class="text-gray-600">electronics</p>
            <p class="mt-2 text-lg font-bold">$1299</p>
          </div>
          <!-- More products... -->
        </div>
      </main>
    </div>
  </div>
</body>
</html>
```

**Network Tab Analysis**:
- Filter: Doc (HTML document)
- Observation: Single request to `/products`
- Timeline:
  - 0-2200ms: Server fetching data and rendering HTML
  - 2200ms: HTML with full content arrives
- Pattern: No client-side fetch requests—all data is in the HTML
- Total time to content: 2.2 seconds (same as before, but...)

**Performance Metrics**:
- **Time to First Byte (TTFB)**: 2250ms (server does the work)
- **First Contentful Paint (FCP)**: 2300ms (content in first paint!)
- **Largest Contentful Paint (LCP)**: 2350ms ✅ (50ms after FCP)
- **Time to Interactive (TTI)**: 2400ms
- **Total Blocking Time (TBT)**: 20ms (minimal hydration)

### Expected vs. Actual Improvement

**Before** (Client Component):
- User sees: Loading spinner → Wait 2.2s → Content appears
- HTML: Empty `<div id="root"></div>`
- JavaScript: 150KB bundle with React + component code
- SEO: Search engines see empty page

**After** (Server Component):
- User sees: Content appears immediately (after server processing)
- HTML: Full content in the initial response
- JavaScript: 45KB bundle (no data fetching code, no component code)
- SEO: Search engines see full content

**Key insight**: The total time is similar (2.2s), but the **user experience** is dramatically different. Instead of staring at a spinner, users see content immediately. The work moved from the client to the server.

### But Wait—We Still Have a Waterfall!

Look at our server-side code again:

```tsx
const products = await getProducts();      // 800ms
const categories = await getCategories();  // 600ms (waits for products)
const featured = await getFeaturedProducts(); // 700ms (waits for categories)
// Total: 2100ms
```

These fetches are still sequential! We're just doing the waterfall on the server instead of the client. Let's fix that.

### Iteration 2: Parallel Server-Side Fetching

We can use `Promise.all()` to fetch all data in parallel:

```tsx
// src/app/products/page.tsx - Parallel fetching
import { getProducts, getCategories, getFeaturedProducts } from '@/lib/api';

export default async function ProductsPage() {
  // Fetch all data in parallel
  const [products, categories, featured] = await Promise.all([
    getProducts(),
    getCategories(),
    getFeaturedProducts(),
  ]);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      
      <div className="grid grid-cols-12 gap-8">
        <aside className="col-span-3">
          <h2 className="text-xl font-semibold mb-4">Categories</h2>
          <ul className="space-y-2">
            {categories.map(cat => (
              <li key={cat.id}>
                {cat.name} ({cat.count})
              </li>
            ))}
          </ul>
        </aside>

        <main className="col-span-9">
          {featured.length > 0 && (
            <div className="mb-8 p-6 bg-blue-50 rounded-lg">
              <h2 className="text-2xl font-semibold mb-4">Featured Products</h2>
              <div className="grid grid-cols-3 gap-4">
                {featured.map(product => (
                  <div key={product.id} className="text-center">
                    <img src={product.imageUrl} alt={product.name} className="w-full h-32 object-cover rounded" />
                    <p className="mt-2 font-medium">{product.name}</p>
                    <p className="text-blue-600">${product.price}</p>
                  </div>
                ))}
              </div>
            </div>
          )}

          <div className="grid grid-cols-3 gap-6">
            {products.map(product => (
              <div key={product.id} className="border rounded-lg p-4">
                <img src={product.imageUrl} alt={product.name} className="w-full h-48 object-cover rounded" />
                <h3 className="mt-4 font-semibold">{product.name}</h3>
                <p className="text-gray-600">{product.category}</p>
                <p className="mt-2 text-lg font-bold">${product.price}</p>
              </div>
            ))}
          </div>
        </main>
      </div>
    </div>
  );
}
```

### Verification: Parallel Fetching Performance

**Server Logs** (add timing to see the difference):

```typescript
// Add this to your page component temporarily
console.log('[Server] Starting parallel fetch...');
const start = performance.now();

const [products, categories, featured] = await Promise.all([
  getProducts(),
  getCategories(),
  getFeaturedProducts(),
]);

console.log(`[Server] All data loaded in ${Math.round(performance.now() - start)}ms`);
```

**Terminal Output**:
```bash
[Server] Starting parallel fetch...
[Server] All data loaded in 823ms
```

**Performance Metrics**:
- **Before** (sequential): 2100ms server processing
- **After** (parallel): 823ms server processing (61% faster!)
- **Time to First Byte (TTFB)**: 873ms (down from 2250ms)
- **First Contentful Paint (FCP)**: 923ms (down from 2300ms)
- **Largest Contentful Paint (LCP)**: 973ms ✅ (down from 2350ms)

### Expected vs. Actual Improvement

**Sequential fetching**:
- Server processing: 2100ms
- User sees content: After 2.2s

**Parallel fetching**:
- Server processing: 823ms (fastest request determines total time)
- User sees content: After 0.9s
- Improvement: **59% faster time to content**

### When to Apply This Solution

**What it optimizes for**:
- Faster time to first byte (TTFB)
- Better user experience (content appears sooner)
- Reduced server processing time
- Better SEO (faster page loads)

**What it sacrifices**:
- Slightly more complex code (but minimal)
- All requests must complete before any content is shown

**When to choose this approach**:
- Multiple independent data sources
- Data fetching is the bottleneck
- All data is needed to render the page
- SEO is important

**When to avoid this approach**:
- Data sources have dependencies (one needs results from another)
- Some data is much slower than others (see Section 17.3 for Streaming)
- You want to show partial content while other data loads

### Limitation Preview

This solves the waterfall problem, but we still have an issue: the **entire page** waits for the **slowest request** to complete. If one API call takes 5 seconds, the user sees nothing for 5 seconds.

What if we could show the fast content immediately and stream in the slow content as it arrives? That's what Streaming and Suspense enable (Section 17.3).

## Real-World Server Component Patterns

Let's look at more realistic data fetching scenarios.

### Pattern 1: Database Queries

In production, you'd fetch from a database, not mock APIs:

```typescript
// src/lib/db.ts - Example with Prisma
import { prisma } from './prisma';

export async function getProducts() {
  return await prisma.product.findMany({
    include: {
      category: true,
      images: true,
    },
    orderBy: {
      createdAt: 'desc',
    },
  });
}

export async function getCategories() {
  return await prisma.category.findMany({
    include: {
      _count: {
        select: { products: true },
      },
    },
  });
}
```

```tsx
// src/app/products/page.tsx - Using database queries
import { getProducts, getCategories } from '@/lib/db';

export default async function ProductsPage() {
  const [products, categories] = await Promise.all([
    getProducts(),
    getCategories(),
  ]);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      {/* Same JSX as before */}
    </div>
  );
}
```

### Pattern 2: External API Calls

Fetching from third-party APIs:

```typescript
// src/lib/external-api.ts
export async function getWeatherData(location: string) {
  const response = await fetch(
    `https://api.weather.com/v1/current?location=${location}`,
    {
      headers: {
        'Authorization': `Bearer ${process.env.WEATHER_API_KEY}`,
      },
    }
  );
  
  if (!response.ok) {
    throw new Error('Failed to fetch weather data');
  }
  
  return response.json();
}
```

```tsx
// src/app/dashboard/page.tsx
import { getWeatherData } from '@/lib/external-api';

export default async function DashboardPage() {
  const weather = await getWeatherData('San Francisco');

  return (
    <div>
      <h1>Dashboard</h1>
      <div className="weather-widget">
        <p>Current temperature: {weather.temperature}°F</p>
        <p>Conditions: {weather.conditions}</p>
      </div>
    </div>
  );
}
```

### Pattern 3: Reading Environment Variables Safely

Server Components can access environment variables that should never be exposed to the client:

```typescript
// src/lib/config.ts - Server-only configuration
export const serverConfig = {
  databaseUrl: process.env.DATABASE_URL!,
  apiKey: process.env.SECRET_API_KEY!,
  stripeSecretKey: process.env.STRIPE_SECRET_KEY!,
};

// This code never ships to the client
// If you try to import this in a Client Component, you'll get a build error
```

### Pattern 4: Nested Server Components

Server Components can render other Server Components, each fetching its own data:

```tsx
// src/components/ProductGrid.tsx - Server Component
import { getProducts } from '@/lib/db';

export async function ProductGrid() {
  const products = await getProducts();

  return (
    <div className="grid grid-cols-3 gap-6">
      {products.map(product => (
        <div key={product.id} className="border rounded-lg p-4">
          <img src={product.imageUrl} alt={product.name} className="w-full h-48 object-cover rounded" />
          <h3 className="mt-4 font-semibold">{product.name}</h3>
          <p className="text-gray-600">{product.category}</p>
          <p className="mt-2 text-lg font-bold">${product.price}</p>
        </div>
      ))}
    </div>
  );
}
```

```tsx
// src/components/CategorySidebar.tsx - Server Component
import { getCategories } from '@/lib/db';

export async function CategorySidebar() {
  const categories = await getCategories();

  return (
    <aside className="col-span-3">
      <h2 className="text-xl font-semibold mb-4">Categories</h2>
      <ul className="space-y-2">
        {categories.map(cat => (
          <li key={cat.id}>
            {cat.name} ({cat.count})
          </li>
        ))}
      </ul>
    </aside>
  );
}
```

```tsx
// src/app/products/page.tsx - Composing Server Components
import { ProductGrid } from '@/components/ProductGrid';
import { CategorySidebar } from '@/components/CategorySidebar';

export default async function ProductsPage() {
  // Each component fetches its own data in parallel
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      
      <div className="grid grid-cols-12 gap-8">
        <CategorySidebar />
        <main className="col-span-9">
          <ProductGrid />
        </main>
      </div>
    </div>
  );
}
```

**Key insight**: When you compose Server Components like this, Next.js automatically parallelizes their data fetching. You don't need `Promise.all()`—the framework handles it.

### Common Failure Modes and Their Signatures

#### Symptom: "You're importing a component that needs X. That only works in a Client Component..."

**Browser behavior**:
Build fails, page doesn't load

**Terminal output**:
```bash
Error: You're importing a component that needs useState. That only works in a Client Component but none of its parents are marked with "use client", so they're Server Components by default.

  1 | import { useState } from 'react';
    |          ^^^^^^^^
```

**Root cause**: Trying to use client-only features (hooks, event handlers) in a Server Component

**Solution**: Add `'use client'` directive at the top of the file, or move the interactive logic to a separate Client Component

#### Symptom: "Error: fetch failed" or database connection errors

**Browser behavior**:
500 error page or error boundary

**Server logs**:
```bash
Error: connect ECONNREFUSED 127.0.0.1:5432
    at TCPConnectWrap.afterConnect [as oncomplete]
```

**Root cause**: Database or API not accessible from the server environment

**Solution**: 
- Check environment variables are set correctly
- Verify database is running and accessible
- Check network/firewall rules in production

#### Symptom: Stale data shown on page

**Browser behavior**:
Page shows old data even after database updates

**Root cause**: Page is statically generated at build time, not regenerated on each request

**Solution**: Use dynamic rendering (see Section 17.4) or revalidation (see Section 17.5)

### When to Use Server Components

**Use Server Components when**:
- Fetching data from databases or APIs
- Accessing server-only resources (file system, environment variables)
- Performing expensive computations
- Rendering static content
- SEO is important

**Don't use Server Components when**:
- You need interactivity (event handlers, state)
- You need browser APIs (localStorage, window, document)
- You need React hooks (useState, useEffect, useContext)
- You need real-time updates without page refresh

For those cases, you need Client Components—which we'll cover in the next section.

## Client Components: use React Query

## When Server Components Aren't Enough

Server Components are excellent for initial page loads, but they have a fundamental limitation: **they can't be interactive**. No event handlers, no state, no hooks.

Consider these scenarios in our product catalog:

1. **Search filtering**: User types in a search box, products filter in real-time
2. **Add to cart**: User clicks "Add to Cart", cart count updates without page reload
3. **Infinite scroll**: User scrolls down, more products load automatically
4. **Real-time updates**: Product availability changes while user is browsing

All of these require **Client Components**—components that run in the browser and can respond to user interactions.

### The Problem: Client-Side Data Fetching (Again)

Let's add a search feature to our product catalog. Users should be able to search products without a full page reload.

### Iteration 3: Naive Client-Side Fetching (The Failure)

First, let's see what happens if we use `useEffect` for client-side data fetching:

```tsx
// src/components/ProductSearch.tsx - Naive approach (PROBLEMATIC)
'use client';

import { useState, useEffect } from 'react';
import { Product } from '@/lib/api';

export function ProductSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }

    setIsLoading(true);
    
    fetch(`/api/products/search?q=${query}`)
      .then(res => res.json())
      .then(data => {
        setResults(data);
        setIsLoading(false);
      })
      .catch(error => {
        console.error('Search failed:', error);
        setIsLoading(false);
      });
  }, [query]);

  return (
    <div className="mb-8">
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search products..."
        className="w-full px-4 py-2 border rounded-lg"
      />
      
      {isLoading && <p className="mt-4">Searching...</p>}
      
      {results.length > 0 && (
        <div className="mt-4 grid grid-cols-3 gap-4">
          {results.map(product => (
            <div key={product.id} className="border rounded-lg p-4">
              <h3 className="font-semibold">{product.name}</h3>
              <p className="text-gray-600">${product.price}</p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

### Diagnostic Analysis: Reading the Client-Side Fetching Failure

Let's type "laptop" into the search box and observe what happens.

**Browser Behavior**:
- User types "l" → Request fires
- User types "la" → Another request fires
- User types "lap" → Another request fires
- User types "lapt" → Another request fires
- User types "lapto" → Another request fires
- User types "laptop" → Another request fires
- Result: 6 requests for a single search term

**Browser Console Output**:
```
[No errors, but let's add logging]
```

Let's add console.log to see the request pattern:

```tsx
useEffect(() => {
  if (!query) {
    setResults([]);
    return;
  }

  console.log(`[${new Date().toISOString()}] Fetching results for: "${query}"`);
  setIsLoading(true);
  
  fetch(`/api/products/search?q=${query}`)
    .then(res => res.json())
    .then(data => {
      console.log(`[${new Date().toISOString()}] Received ${data.length} results for: "${query}"`);
      setResults(data);
      setIsLoading(false);
    });
}, [query]);
```

**Browser Console Output**:
```
[2024-01-15T10:23:45.123Z] Fetching results for: "l"
[2024-01-15T10:23:45.234Z] Fetching results for: "la"
[2024-01-15T10:23:45.345Z] Fetching results for: "lap"
[2024-01-15T10:23:45.456Z] Fetching results for: "lapt"
[2024-01-15T10:23:45.567Z] Fetching results for: "lapto"
[2024-01-15T10:23:45.678Z] Fetching results for: "laptop"
[2024-01-15T10:23:45.823Z] Received 0 results for: "l"
[2024-01-15T10:23:45.934Z] Received 0 results for: "la"
[2024-01-15T10:23:46.045Z] Received 3 results for: "lap"
[2024-01-15T10:23:46.156Z] Received 3 results for: "lapt"
[2024-01-15T10:23:46.267Z] Received 3 results for: "lapto"
[2024-01-15T10:23:46.378Z] Received 3 results for: "laptop"
```

**Network Tab Analysis**:
- Filter: Fetch/XHR
- Observation: 6 requests to `/api/products/search` in rapid succession
- Timeline: Requests fire every ~100ms as user types
- Pattern: No debouncing—every keystroke triggers a request
- Total requests: 6 for a 6-character search term
- Wasted bandwidth: First 5 requests are obsolete by the time they complete

**React DevTools Evidence**:
- `ProductSearch` component selected
- State updates: `query` changes 6 times in 0.5 seconds
- Effect runs: 6 times (once per state change)
- Render count: 12+ renders (state changes + loading states)

### Let's Parse This Evidence

1. **What the user experiences**:
   - Expected: Smooth search experience with results appearing as they type
   - Actual: Flickering loading states, results changing rapidly, wasted network requests

2. **What the console reveals**:
   - Key indicator: Requests fire on every keystroke
   - Pattern: No debouncing or request cancellation
   - Problem: Intermediate queries ("l", "la", "lap") are useless

3. **What the Network tab shows**:
   - Technical evidence: Request waterfall with overlapping requests
   - Wasted work: Server processes 6 queries when only the last one matters
   - Performance impact: Unnecessary server load and bandwidth usage

4. **Root cause identified**: 
   No debouncing mechanism—every state change triggers a new fetch, even for incomplete search terms.

5. **Why the current approach can't solve this**:
   Even if we add debouncing, we still have problems:
   - Manual loading state management
   - No error handling
   - No request cancellation (old requests can return after new ones)
   - No caching (same search twice = two requests)
   - No retry logic for failed requests
   - No background refetching for stale data

6. **What we need**:
   A library that handles all these concerns automatically: debouncing, caching, error handling, request cancellation, background updates, and more.

### Additional Problems with Naive Client Fetching

Let's explore more failure modes:

#### Problem 1: Race Conditions

User types "laptop" quickly, then deletes and types "mouse". If the "laptop" request is slow and the "mouse" request is fast, the "laptop" results might arrive last and overwrite the "mouse" results.

```tsx
// Simulating race condition
useEffect(() => {
  if (!query) return;

  setIsLoading(true);
  
  // Simulate variable network latency
  const delay = Math.random() * 2000;
  
  setTimeout(() => {
    fetch(`/api/products/search?q=${query}`)
      .then(res => res.json())
      .then(data => {
        // This might set results for an old query!
        setResults(data);
        setIsLoading(false);
      });
  }, delay);
}, [query]);
```

**Browser Console Output** (race condition):
```
[10:23:45.123Z] Fetching results for: "laptop"
[10:23:46.234Z] Fetching results for: "mouse"  ← User changed their mind
[10:23:46.456Z] Received 5 results for: "mouse" ← Fast request returns first
[10:23:47.789Z] Received 3 results for: "laptop" ← Slow request returns last
```

**Result**: User sees "mouse" results briefly, then they're replaced by "laptop" results—even though the user searched for "mouse"!

#### Problem 2: No Caching

User searches for "laptop", sees results, searches for "mouse", then searches for "laptop" again. The second "laptop" search makes another network request, even though we already have that data.

#### Problem 3: No Error Recovery

Network request fails—now what? Show an error message? Retry automatically? How many times? With exponential backoff?

All of this is complex to implement correctly. This is where **React Query** (TanStack Query) comes in.

## React Query: Professional Client-Side Data Fetching

React Query is a library that solves all the problems we just identified:

- ✅ Automatic caching
- ✅ Background refetching
- ✅ Request deduplication
- ✅ Automatic retries with exponential backoff
- ✅ Loading and error states
- ✅ Optimistic updates
- ✅ Pagination and infinite scroll
- ✅ Request cancellation
- ✅ DevTools for debugging

### Installation

```bash
npm install @tanstack/react-query
```

### Setup: Query Client Provider

React Query requires a provider at the root of your app:

```tsx
// src/app/providers.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  // Create a client instance
  // Use useState to ensure it's only created once per component mount
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // Data is fresh for 1 minute
        refetchOnWindowFocus: false, // Don't refetch when user returns to tab
      },
    },
  }));

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

```tsx
// src/app/layout.tsx
import { Providers } from './providers';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <Providers>
          {children}
        </Providers>
      </body>
    </html>
  );
}
```

### Iteration 4: Search with React Query

Now let's rebuild our search component using React Query:

```tsx
// src/components/ProductSearch.tsx - React Query approach
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { Product } from '@/lib/api';
import { useDebounce } from '@/hooks/useDebounce';

async function searchProducts(query: string): Promise<Product[]> {
  if (!query) return [];
  
  const response = await fetch(`/api/products/search?q=${query}`);
  if (!response.ok) {
    throw new Error('Search failed');
  }
  return response.json();
}

export function ProductSearch() {
  const [query, setQuery] = useState('');
  
  // Debounce the query to avoid excessive requests
  const debouncedQuery = useDebounce(query, 300);

  // React Query handles caching, loading states, errors, and more
  const { data: results = [], isLoading, error } = useQuery({
    queryKey: ['products', 'search', debouncedQuery],
    queryFn: () => searchProducts(debouncedQuery),
    enabled: debouncedQuery.length > 0, // Only run query if there's a search term
  });

  return (
    <div className="mb-8">
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search products..."
        className="w-full px-4 py-2 border rounded-lg"
      />
      
      {isLoading && <p className="mt-4 text-gray-600">Searching...</p>}
      
      {error && (
        <p className="mt-4 text-red-600">
          Search failed. Please try again.
        </p>
      )}
      
      {results.length > 0 && (
        <div className="mt-4 grid grid-cols-3 gap-4">
          {results.map(product => (
            <div key={product.id} className="border rounded-lg p-4">
              <h3 className="font-semibold">{product.name}</h3>
              <p className="text-gray-600">${product.price}</p>
            </div>
          ))}
        </div>
      )}
      
      {debouncedQuery && results.length === 0 && !isLoading && (
        <p className="mt-4 text-gray-600">No products found.</p>
      )}
    </div>
  );
}
```

```typescript
// src/hooks/useDebounce.ts - Debounce hook
import { useEffect, useState } from 'react';

export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);

  return debouncedValue;
}
```

### What Changed?

**Before** (manual useEffect):
```tsx
const [results, setResults] = useState<Product[]>([]);
const [isLoading, setIsLoading] = useState(false);

useEffect(() => {
  setIsLoading(true);
  fetch(`/api/products/search?q=${query}`)
    .then(res => res.json())
    .then(data => {
      setResults(data);
      setIsLoading(false);
    });
}, [query]);
```

**After** (React Query):
```tsx
const { data: results = [], isLoading, error } = useQuery({
  queryKey: ['products', 'search', debouncedQuery],
  queryFn: () => searchProducts(debouncedQuery),
  enabled: debouncedQuery.length > 0,
});
```

### Verification: React Query in Action

Let's type "laptop" again and observe the difference.

**Browser Console Output**:
```
[10:23:45.123Z] User typed: "l"
[10:23:45.234Z] User typed: "la"
[10:23:45.345Z] User typed: "lap"
[10:23:45.456Z] User typed: "lapt"
[10:23:45.567Z] User typed: "lapto"
[10:23:45.678Z] User typed: "laptop"
[10:23:45.978Z] Debounce complete, fetching results for: "laptop"
[10:23:46.189Z] Received 3 results for: "laptop"
```

**Network Tab Analysis**:
- Filter: Fetch/XHR
- Observation: **Only 1 request** to `/api/products/search?q=laptop`
- Timeline: Request fires 300ms after user stops typing
- Pattern: Debouncing works—intermediate keystrokes don't trigger requests
- Total requests: 1 (down from 6!)

**React Query DevTools** (open the floating icon in bottom-right):
- Query key: `['products', 'search', 'laptop']`
- Status: `success`
- Data: Array of 3 products
- Last updated: 2 seconds ago
- Stale in: 58 seconds (based on our `staleTime` config)

### Expected vs. Actual Improvement

**Before** (manual useEffect):
- Requests: 6 (one per keystroke)
- Caching: None (same search twice = two requests)
- Error handling: Manual try/catch
- Loading states: Manual state management
- Race conditions: Possible
- Code complexity: High

**After** (React Query):
- Requests: 1 (debounced)
- Caching: Automatic (same search twice = instant from cache)
- Error handling: Built-in with `error` state
- Loading states: Built-in with `isLoading` state
- Race conditions: Impossible (React Query cancels old requests)
- Code complexity: Low

### Demonstrating the Cache

Let's prove the caching works:

1. Search for "laptop" → Request fires, results appear
2. Clear the search box
3. Search for "laptop" again → **No request fires**, results appear instantly from cache

**React Query DevTools** shows:
- Query status: `success` (from cache)
- Data age: 5 seconds
- No network request in Network tab

### React Query Core Concepts

#### 1. Query Keys

Query keys uniquely identify queries for caching:

```typescript
// Simple key
useQuery({
  queryKey: ['products'],
  queryFn: getProducts,
});

// Key with parameters
useQuery({
  queryKey: ['products', 'search', query],
  queryFn: () => searchProducts(query),
});

// Key with multiple parameters
useQuery({
  queryKey: ['products', { category: 'electronics', sort: 'price' }],
  queryFn: () => getProducts({ category: 'electronics', sort: 'price' }),
});
```

**Rule**: If the query key changes, React Query treats it as a different query and fetches new data.

#### 2. Query Functions

The function that actually fetches the data:

```typescript
// Simple fetch
const queryFn = () => fetch('/api/products').then(res => res.json());

// With parameters from query key
const queryFn = ({ queryKey }) => {
  const [_key, _search, query] = queryKey;
  return searchProducts(query);
};

// Async function
const queryFn = async () => {
  const response = await fetch('/api/products');
  if (!response.ok) {
    throw new Error('Failed to fetch products');
  }
  return response.json();
};
```

#### 3. Query Options

Configure query behavior:

```typescript
useQuery({
  queryKey: ['products'],
  queryFn: getProducts,
  
  // Only run query if condition is true
  enabled: isLoggedIn,
  
  // How long data stays fresh (no refetch during this time)
  staleTime: 5 * 60 * 1000, // 5 minutes
  
  // How long unused data stays in cache
  cacheTime: 10 * 60 * 1000, // 10 minutes
  
  // Retry failed requests
  retry: 3,
  
  // Exponential backoff between retries
  retryDelay: attemptIndex => Math.min(1000 * 2 ** attemptIndex, 30000),
  
  // Refetch on window focus
  refetchOnWindowFocus: true,
  
  // Refetch on network reconnect
  refetchOnReconnect: true,
});
```

### Real-World React Query Patterns

#### Pattern 1: Dependent Queries

One query depends on the result of another:

```tsx
// src/components/ProductDetails.tsx
'use client';

import { useQuery } from '@tanstack/react-query';

export function ProductDetails({ productId }: { productId: string }) {
  // First query: Get product details
  const { data: product } = useQuery({
    queryKey: ['products', productId],
    queryFn: () => getProduct(productId),
  });

  // Second query: Get related products (depends on first query)
  const { data: relatedProducts } = useQuery({
    queryKey: ['products', 'related', product?.category],
    queryFn: () => getRelatedProducts(product!.category),
    enabled: !!product, // Only run when product is loaded
  });

  if (!product) return <div>Loading...</div>;

  return (
    <div>
      <h1>{product.name}</h1>
      <p>${product.price}</p>
      
      {relatedProducts && (
        <div className="mt-8">
          <h2>Related Products</h2>
          {relatedProducts.map(p => (
            <div key={p.id}>{p.name}</div>
          ))}
        </div>
      )}
    </div>
  );
}
```

#### Pattern 2: Pagination

Fetching paginated data:

```tsx
// src/components/ProductList.tsx
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';

export function ProductList() {
  const [page, setPage] = useState(1);

  const { data, isLoading, isPlaceholderData } = useQuery({
    queryKey: ['products', 'list', page],
    queryFn: () => getProducts({ page, limit: 20 }),
    placeholderData: (previousData) => previousData, // Keep old data while fetching new
  });

  return (
    <div>
      <div className="grid grid-cols-3 gap-4">
        {data?.products.map(product => (
          <div key={product.id}>{product.name}</div>
        ))}
      </div>
      
      <div className="mt-8 flex gap-4">
        <button
          onClick={() => setPage(p => Math.max(1, p - 1))}
          disabled={page === 1}
        >
          Previous
        </button>
        
        <span>Page {page}</span>
        
        <button
          onClick={() => setPage(p => p + 1)}
          disabled={isPlaceholderData || !data?.hasMore}
        >
          Next
        </button>
      </div>
    </div>
  );
}
```

#### Pattern 3: Infinite Scroll

Loading more data as user scrolls:

```tsx
// src/components/InfiniteProductList.tsx
'use client';

import { useInfiniteQuery } from '@tanstack/react-query';
import { useEffect, useRef } from 'react';

export function InfiniteProductList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['products', 'infinite'],
    queryFn: ({ pageParam = 1 }) => getProducts({ page: pageParam, limit: 20 }),
    getNextPageParam: (lastPage, pages) => {
      return lastPage.hasMore ? pages.length + 1 : undefined;
    },
    initialPageParam: 1,
  });

  // Intersection Observer for infinite scroll
  const loadMoreRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!loadMoreRef.current) return;

    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      },
      { threshold: 1.0 }
    );

    observer.observe(loadMoreRef.current);

    return () => observer.disconnect();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (
    <div>
      <div className="grid grid-cols-3 gap-4">
        {data?.pages.map((page, i) => (
          <div key={i}>
            {page.products.map(product => (
              <div key={product.id}>{product.name}</div>
            ))}
          </div>
        ))}
      </div>
      
      <div ref={loadMoreRef} className="h-20 flex items-center justify-center">
        {isFetchingNextPage && <p>Loading more...</p>}
      </div>
    </div>
  );
}
```

#### Pattern 4: Mutations (Creating/Updating Data)

React Query also handles mutations (POST, PUT, DELETE):

```tsx
// src/components/AddToCartButton.tsx
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';

async function addToCart(productId: string) {
  const response = await fetch('/api/cart', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ productId }),
  });
  
  if (!response.ok) {
    throw new Error('Failed to add to cart');
  }
  
  return response.json();
}

export function AddToCartButton({ productId }: { productId: string }) {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: () => addToCart(productId),
    onSuccess: () => {
      // Invalidate cart query to refetch
      queryClient.invalidateQueries({ queryKey: ['cart'] });
    },
  });

  return (
    <button
      onClick={() => mutation.mutate()}
      disabled={mutation.isPending}
      className="px-4 py-2 bg-blue-600 text-white rounded"
    >
      {mutation.isPending ? 'Adding...' : 'Add to Cart'}
    </button>
  );
}
```

### Common Failure Modes and Their Signatures

#### Symptom: "No QueryClient set, use QueryClientProvider to set one"

**Browser behavior**:
Error boundary or blank page

**Browser Console Output**:
```
Error: No QueryClient set, use QueryClientProvider to set one
    at useQueryClient (react-query.js:123)
```

**Root cause**: Forgot to wrap app in `QueryClientProvider`

**Solution**: Add provider in root layout (see setup section above)

#### Symptom: Query refetches on every render

**Browser behavior**:
Excessive network requests, poor performance

**React Query DevTools**:
- Query status constantly switching between `fetching` and `success`
- Fetch count incrementing rapidly

**Root cause**: Query key includes unstable reference (object/array created inline)

**Solution**: Memoize query key or use primitive values

```tsx
// ❌ Bad: Object created on every render
useQuery({
  queryKey: ['products', { category: 'electronics' }], // New object each time!
  queryFn: getProducts,
});

// ✅ Good: Stable primitive values
const category = 'electronics';
useQuery({
  queryKey: ['products', category],
  queryFn: () => getProducts({ category }),
});
```

#### Symptom: Stale data shown after mutation

**Browser behavior**:
User adds item to cart, but cart count doesn't update

**Root cause**: Forgot to invalidate related queries after mutation

**Solution**: Use `queryClient.invalidateQueries()` in mutation's `onSuccess`

```tsx
const mutation = useMutation({
  mutationFn: addToCart,
  onSuccess: () => {
    // Invalidate all queries that start with ['cart']
    queryClient.invalidateQueries({ queryKey: ['cart'] });
  },
});
```

### When to Apply This Solution

**What React Query optimizes for**:
- Automatic caching and background updates
- Simplified loading and error states
- Request deduplication and cancellation
- Optimistic updates
- Developer experience (less boilerplate)

**What it sacrifices**:
- Additional bundle size (~13KB gzipped)
- Learning curve for advanced features
- Another dependency to maintain

**When to choose React Query**:
- Complex client-side data fetching requirements
- Need caching, background updates, or optimistic updates
- Multiple components fetching the same data
- Pagination or infinite scroll
- Real-time data that needs periodic refetching

**When to avoid React Query**:
- Simple one-time fetches (use Server Components instead)
- All data can be fetched on the server
- Bundle size is critical and you can't afford 13KB
- Team is unfamiliar and timeline is tight

**Code characteristics**:
- Setup: Medium (provider + configuration)
- Maintenance: Low (library handles complexity)
- Performance: Excellent (automatic optimizations)

### Limitation Preview

React Query solves client-side data fetching, but we still have a problem: the **entire page** waits for the **slowest Server Component** to finish rendering before any HTML is sent to the client.

What if we could send the fast parts of the page immediately and stream in the slow parts as they become ready? That's what Streaming and Suspense enable (next section).

## Streaming and Suspense

## The Problem: Slow Components Block the Entire Page

Let's return to our product catalog. We have three data sources with different speeds:

1. **Categories** (fast): 100ms - cached in Redis
2. **Products** (medium): 500ms - database query
3. **Recommendations** (slow): 3000ms - ML model inference

With our current Server Component approach, the page waits for **all three** to complete before sending any HTML to the client.

### Iteration 5: The All-or-Nothing Problem (The Failure)

Let's add a recommendations section that's intentionally slow:

```typescript
// src/lib/api.ts - Add slow recommendations
export async function getRecommendations(userId: string): Promise<Product[]> {
  // Simulate ML model inference - very slow
  await new Promise(resolve => setTimeout(resolve, 3000));
  
  return [
    { id: '10', name: 'Recommended Item 1', price: 199, category: 'electronics', imageUrl: '/rec1.jpg', featured: false },
    { id: '11', name: 'Recommended Item 2', price: 299, category: 'electronics', imageUrl: '/rec2.jpg', featured: false },
  ];
}
```

```tsx
// src/app/products/page.tsx - With slow recommendations
import { getProducts, getCategories, getRecommendations } from '@/lib/api';

export default async function ProductsPage() {
  // All fetches run in parallel, but page waits for slowest
  const [products, categories, recommendations] = await Promise.all([
    getProducts(),      // 500ms
    getCategories(),    // 100ms
    getRecommendations('user-123'), // 3000ms ← Blocks everything!
  ]);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      
      <div className="grid grid-cols-12 gap-8">
        {/* Categories - ready in 100ms, but waits 3000ms to show */}
        <aside className="col-span-3">
          <h2 className="text-xl font-semibold mb-4">Categories</h2>
          <ul className="space-y-2">
            {categories.map(cat => (
              <li key={cat.id}>{cat.name} ({cat.count})</li>
            ))}
          </ul>
        </aside>

        <main className="col-span-9">
          {/* Products - ready in 500ms, but waits 3000ms to show */}
          <div className="grid grid-cols-3 gap-6">
            {products.map(product => (
              <div key={product.id} className="border rounded-lg p-4">
                <h3 className="font-semibold">{product.name}</h3>
                <p className="text-gray-600">${product.price}</p>
              </div>
            ))}
          </div>

          {/* Recommendations - takes 3000ms */}
          <div className="mt-8">
            <h2 className="text-2xl font-semibold mb-4">Recommended for You</h2>
            <div className="grid grid-cols-3 gap-4">
              {recommendations.map(product => (
                <div key={product.id} className="border rounded-lg p-4">
                  <h3 className="font-semibold">{product.name}</h3>
                  <p className="text-gray-600">${product.price}</p>
                </div>
              ))}
            </div>
          </div>
        </main>
      </div>
    </div>
  );
}
```

### Diagnostic Analysis: Reading the Blocking Failure

Let's navigate to `/products` and observe what happens.

**Browser Behavior**:
- User navigates to `/products`
- Browser shows loading indicator for 3+ seconds
- Then entire page appears at once
- No progressive loading—everything or nothing

**Network Tab Analysis**:
- Filter: Doc (HTML document)
- Observation: Single request to `/products`
- Timeline:
  - 0-3000ms: Waiting for server response (TTFB)
  - 3000ms: HTML arrives with all content
- Pattern: Server holds the response until all data is ready
- Total time to content: 3+ seconds

**Server Logs** (add timing):

```typescript
console.log('[Server] Starting data fetch...');
const start = performance.now();

const [products, categories, recommendations] = await Promise.all([
  getProducts(),
  getCategories(),
  getRecommendations('user-123'),
]);

console.log(`[Server] Data ready in ${Math.round(performance.now() - start)}ms`);
console.log(`[Server] Rendering HTML...`);
```

**Terminal Output**:
```bash
[Server] Starting data fetch...
[Server] Data ready in 3012ms
[Server] Rendering HTML...
```

**Performance Metrics**:
- **Time to First Byte (TTFB)**: 3050ms ⚠️ (terrible!)
- **First Contentful Paint (FCP)**: 3100ms ⚠️
- **Largest Contentful Paint (LCP)**: 3150ms ⚠️
- **Time to Interactive (TTI)**: 3200ms ⚠️

### Let's Parse This Evidence

1. **What the user experiences**:
   - Expected: See fast content (categories, products) immediately, recommendations load later
   - Actual: Stare at blank page for 3 seconds, then everything appears at once

2. **What the Network tab shows**:
   - Key indicator: TTFB is 3 seconds—server is holding the response
   - Pattern: All-or-nothing—no progressive rendering
   - Wasted opportunity: Categories and products are ready in 500ms, but user doesn't see them

3. **What the server logs reveal**:
   - Technical evidence: `Promise.all()` waits for slowest promise (3000ms)
   - Root cause: Synchronous rendering—page can't be sent until all data is ready

4. **Root cause identified**: 
   Server Components render synchronously. The page waits for all async operations to complete before sending any HTML to the client.

5. **Why the current approach can't solve this**:
   Even if we optimize the recommendations query, we'll always have this problem:
   - Any slow component blocks the entire page
   - Users see nothing while waiting for slow data
   - Fast content is held hostage by slow content
   - No way to show partial results

6. **What we need**:
   A way to send the fast parts of the page immediately and stream in the slow parts as they become ready. This is what **Streaming** and **Suspense** enable.

### The Fundamental Problem: Synchronous Rendering

Traditional server rendering is all-or-nothing:

```
Server: Fetch all data → Render all HTML → Send complete response
Client: Wait... wait... wait... → Display everything at once
```

What we want:

```
Server: Fetch fast data → Send partial HTML → Continue fetching slow data → Send more HTML
Client: Display fast content → Show loading state → Display slow content when ready
```

This is **streaming**—sending HTML in chunks as it becomes ready.

## Streaming with Suspense

React 18 introduced **Suspense** for Server Components, enabling streaming. Here's how it works:

1. Wrap slow components in `<Suspense>` with a fallback
2. Next.js sends the fast parts of the page immediately
3. Slow components show the fallback (loading state)
4. When slow data is ready, Next.js streams the real content
5. React replaces the fallback with the real content (no page reload)

### Iteration 6: Streaming with Suspense

Let's refactor our page to stream the slow recommendations:

```tsx
// src/components/RecommendationsSection.tsx - Slow component extracted
import { getRecommendations } from '@/lib/api';

export async function RecommendationsSection({ userId }: { userId: string }) {
  // This is slow (3000ms), but won't block the page
  const recommendations = await getRecommendations(userId);

  return (
    <div className="mt-8">
      <h2 className="text-2xl font-semibold mb-4">Recommended for You</h2>
      <div className="grid grid-cols-3 gap-4">
        {recommendations.map(product => (
          <div key={product.id} className="border rounded-lg p-4">
            <h3 className="font-semibold">{product.name}</h3>
            <p className="text-gray-600">${product.price}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

```tsx
// src/app/products/page.tsx - With Suspense
import { Suspense } from 'react';
import { getProducts, getCategories } from '@/lib/api';
import { RecommendationsSection } from '@/components/RecommendationsSection';

export default async function ProductsPage() {
  // Only fetch fast data here
  const [products, categories] = await Promise.all([
    getProducts(),    // 500ms
    getCategories(),  // 100ms
  ]);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      
      <div className="grid grid-cols-12 gap-8">
        {/* Categories - shows immediately */}
        <aside className="col-span-3">
          <h2 className="text-xl font-semibold mb-4">Categories</h2>
          <ul className="space-y-2">
            {categories.map(cat => (
              <li key={cat.id}>{cat.name} ({cat.count})</li>
            ))}
          </ul>
        </aside>

        <main className="col-span-9">
          {/* Products - shows immediately */}
          <div className="grid grid-cols-3 gap-6">
            {products.map(product => (
              <div key={product.id} className="border rounded-lg p-4">
                <h3 className="font-semibold">{product.name}</h3>
                <p className="text-gray-600">${product.price}</p>
              </div>
            ))}
          </div>

          {/* Recommendations - streams in later */}
          <Suspense fallback={<RecommendationsLoading />}>
            <RecommendationsSection userId="user-123" />
          </Suspense>
        </main>
      </div>
    </div>
  );
}

function RecommendationsLoading() {
  return (
    <div className="mt-8">
      <h2 className="text-2xl font-semibold mb-4">Recommended for You</h2>
      <div className="grid grid-cols-3 gap-4">
        {[1, 2, 3].map(i => (
          <div key={i} className="border rounded-lg p-4 animate-pulse">
            <div className="h-4 bg-gray-200 rounded w-3/4 mb-2"></div>
            <div className="h-4 bg-gray-200 rounded w-1/2"></div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### What Changed?

**Before** (blocking):
```tsx
const [products, categories, recommendations] = await Promise.all([
  getProducts(),
  getCategories(),
  getRecommendations('user-123'), // Blocks everything!
]);

return (
  <div>
    {/* All content rendered together */}
    <Categories data={categories} />
    <Products data={products} />
    <Recommendations data={recommendations} />
  </div>
);
```

**After** (streaming):
```tsx
// Only fetch fast data
const [products, categories] = await Promise.all([
  getProducts(),
  getCategories(),
]);

return (
  <div>
    {/* Fast content rendered immediately */}
    <Categories data={categories} />
    <Products data={products} />
    
    {/* Slow content wrapped in Suspense */}
    <Suspense fallback={<Loading />}>
      <RecommendationsSection userId="user-123" />
    </Suspense>
  </div>
);
```

### Verification: Streaming in Action

Let's navigate to `/products` and observe the streaming behavior.

**Browser Behavior** (watch carefully):
1. Page appears immediately with categories and products (500ms)
2. Recommendations section shows loading skeleton
3. After 3 seconds, loading skeleton is replaced with real recommendations
4. No page reload—content streams in seamlessly

**Network Tab Analysis**:
- Filter: Doc (HTML document)
- Observation: Single request to `/products`, but response arrives in **chunks**
- Timeline:
  - 0-500ms: Waiting for fast data
  - 500ms: First chunk arrives (HTML with categories, products, and loading fallback)
  - 500-3500ms: Connection stays open
  - 3500ms: Second chunk arrives (HTML with recommendations)
- Pattern: **Streaming response**—HTML sent in multiple chunks
- Total time to first content: 500ms (down from 3000ms!)
- Total time to complete: 3500ms (same as before, but user sees content sooner)

**Performance Metrics**:
- **Time to First Byte (TTFB)**: 550ms ✅ (down from 3050ms!)
- **First Contentful Paint (FCP)**: 600ms ✅ (down from 3100ms!)
- **Largest Contentful Paint (LCP)**: 650ms ✅ (down from 3150ms!)
- **Time to Interactive (TTI)**: 700ms ✅ (down from 3200ms!)

### Expected vs. Actual Improvement

**Before** (blocking):
- User sees: Nothing → Wait 3s → Everything appears
- TTFB: 3050ms
- FCP: 3100ms
- User perception: "This site is slow"

**After** (streaming):
- User sees: Categories + Products (500ms) → Loading skeleton → Recommendations (3500ms)
- TTFB: 550ms (82% faster!)
- FCP: 600ms (81% faster!)
- User perception: "This site is fast, recommendations are loading"

**Key insight**: The total time to complete is similar, but the **perceived performance** is dramatically better. Users see content immediately instead of staring at a blank page.

### How Streaming Works Under the Hood

Let's look at the actual HTML that gets streamed:

**First Chunk** (arrives at 500ms):
```html
<!DOCTYPE html>
<html>
<body>
  <div class="container">
    <h1>Our Products</h1>
    
    <!-- Categories and products are here -->
    <aside>
      <h2>Categories</h2>
      <ul>
        <li>Electronics (45)</li>
        <li>Furniture (23)</li>
      </ul>
    </aside>
    
    <main>
      <div class="grid">
        <div>Laptop Pro - $1299</div>
        <!-- More products... -->
      </div>
      
      <!-- Suspense fallback -->
      <div id="recommendations-fallback">
        <h2>Recommended for You</h2>
        <div class="animate-pulse">Loading...</div>
      </div>
    </main>
  </div>
  
  <!-- Script to handle streaming -->
  <script>/* React streaming runtime */</script>
</body>
</html>
```

**Second Chunk** (arrives at 3500ms):
```html
<!-- Streamed content -->
<template id="recommendations-content">
  <div class="mt-8">
    <h2>Recommended for You</h2>
    <div class="grid">
      <div>Recommended Item 1 - $199</div>
      <div>Recommended Item 2 - $299</div>
    </div>
  </div>
</template>

<script>
  // React replaces fallback with real content
  const fallback = document.getElementById('recommendations-fallback');
  const content = document.getElementById('recommendations-content');
  fallback.replaceWith(content.content);
</script>
```

React handles all this automatically—you just wrap slow components in `<Suspense>`.

## Advanced Streaming Patterns

### Pattern 1: Multiple Suspense Boundaries

You can have multiple independent streaming sections:

```tsx
// src/app/dashboard/page.tsx - Multiple streaming sections
import { Suspense } from 'react';

export default function DashboardPage() {
  return (
    <div className="grid grid-cols-2 gap-8">
      {/* Left column - fast */}
      <div>
        <h2>Quick Stats</h2>
        <QuickStats />
      </div>

      {/* Right column - multiple slow sections */}
      <div>
        <Suspense fallback={<ChartLoading />}>
          <RevenueChart /> {/* 2 seconds */}
        </Suspense>

        <Suspense fallback={<TableLoading />}>
          <RecentOrders /> {/* 1 second */}
        </Suspense>

        <Suspense fallback={<ListLoading />}>
          <TopProducts /> {/* 3 seconds */}
        </Suspense>
      </div>
    </div>
  );
}
```

**Streaming timeline**:
- 0ms: Page structure and QuickStats appear
- 1000ms: RecentOrders streams in
- 2000ms: RevenueChart streams in
- 3000ms: TopProducts streams in

Each section streams independently—fast sections don't wait for slow ones.

### Pattern 2: Nested Suspense

Suspense boundaries can be nested for fine-grained control:

```tsx
// src/app/product/[id]/page.tsx - Nested Suspense
import { Suspense } from 'react';

export default function ProductPage({ params }: { params: { id: string } }) {
  return (
    <div>
      {/* Outer Suspense - entire product section */}
      <Suspense fallback={<ProductPageLoading />}>
        <ProductDetails productId={params.id}>
          {/* Inner Suspense - just reviews */}
          <Suspense fallback={<ReviewsLoading />}>
            <ProductReviews productId={params.id} />
          </Suspense>
        </ProductDetails>
      </Suspense>
    </div>
  );
}
```

**Streaming timeline**:
- 0ms: Page structure appears with outer loading state
- 500ms: Product details stream in, reviews show loading state
- 2000ms: Reviews stream in

### Pattern 3: Conditional Suspense

Only wrap in Suspense if the component is actually slow:

```tsx
// src/app/products/page.tsx - Conditional Suspense
export default async function ProductsPage({
  searchParams,
}: {
  searchParams: { premium?: string };
}) {
  const isPremiumUser = searchParams.premium === 'true';

  return (
    <div>
      <ProductGrid />

      {/* Only show recommendations for premium users */}
      {isPremiumUser && (
        <Suspense fallback={<RecommendationsLoading />}>
          <PersonalizedRecommendations />
        </Suspense>
      )}
    </div>
  );
}
```

### Pattern 4: Preloading Data

Start fetching data before it's needed:

```typescript
// src/lib/preload.ts
import { cache } from 'react';

// Cache ensures the same data isn't fetched twice
export const getProduct = cache(async (id: string) => {
  const response = await fetch(`/api/products/${id}`);
  return response.json();
});

export function preloadProduct(id: string) {
  // Start fetching, but don't await
  void getProduct(id);
}
```

```tsx
// src/app/products/page.tsx - Preload on hover
'use client';

import { preloadProduct } from '@/lib/preload';

export function ProductCard({ product }) {
  return (
    <Link
      href={`/products/${product.id}`}
      onMouseEnter={() => preloadProduct(product.id)}
    >
      <h3>{product.name}</h3>
    </Link>
  );
}
```

### Common Failure Modes and Their Signatures

#### Symptom: Suspense boundary never resolves

**Browser behavior**:
Loading fallback shows forever, content never appears

**Browser Console Output**:
```
Warning: A component suspended while responding to synchronous input.
This will cause the UI to be replaced with a loading indicator.
```

**Root cause**: Component inside Suspense is throwing an error, not suspending properly

**Solution**: Check server logs for errors, ensure async component is actually awaiting promises

#### Symptom: Content flashes (fallback → content → fallback → content)

**Browser behavior**:
Loading state appears briefly, then content, then loading again

**Root cause**: Component is re-fetching data on every render (no caching)

**Solution**: Use React's `cache()` function to deduplicate requests

```typescript
// ❌ Bad: Fetches on every render
export async function ProductDetails({ id }: { id: string }) {
  const product = await fetch(`/api/products/${id}`).then(r => r.json());
  return <div>{product.name}</div>;
}

// ✅ Good: Cached, only fetches once
import { cache } from 'react';

const getProduct = cache(async (id: string) => {
  return fetch(`/api/products/${id}`).then(r => r.json());
});

export async function ProductDetails({ id }: { id: string }) {
  const product = await getProduct(id);
  return <div>{product.name}</div>;
}
```

#### Symptom: Entire page waits for Suspense boundary

**Browser behavior**:
No streaming—page behaves like before (all-or-nothing)

**Root cause**: Page is statically generated at build time, not dynamically rendered

**Solution**: Force dynamic rendering (see next section)

### When to Apply This Solution

**What Suspense optimizes for**:
- Perceived performance (fast content shows immediately)
- Progressive rendering (show what you have, load the rest)
- Better user experience (no blank page staring)
- Flexibility (independent loading states)

**What it sacrifices**:
- Slightly more complex component structure
- Need to design good loading states
- Debugging can be harder (multiple render passes)

**When to choose Suspense**:
- Page has mix of fast and slow data sources
- Some content is much slower than others
- User experience is critical
- You want to show partial results

**When to avoid Suspense**:
- All data is equally fast
- Page is simple and loads quickly
- Loading states would be distracting
- You need all data before showing anything (e.g., checkout page)

**Code characteristics**:
- Setup: Low (just wrap in `<Suspense>`)
- Maintenance: Low (React handles streaming)
- Performance: Excellent (progressive rendering)

### Limitation Preview

Suspense solves progressive rendering, but we still have a question: should this page be **statically generated** at build time or **dynamically rendered** on each request?

The answer depends on how often the data changes and whether it's personalized. That's what we'll explore in the next section.

## Static vs. dynamic rendering

## The Question: Build Time or Request Time?

Next.js can render pages in two fundamentally different ways:

1. **Static Rendering** (SSG): Generate HTML at build time, serve the same HTML to all users
2. **Dynamic Rendering** (SSR): Generate HTML on each request, personalized for each user

The choice dramatically affects performance, scalability, and user experience.

### The Mental Model

**Static Rendering**:
```
Build time: Fetch data → Render HTML → Save to disk
Request time: Serve pre-built HTML (instant!)
```

**Dynamic Rendering**:
```
Build time: Nothing
Request time: Fetch data → Render HTML → Send to user (slower, but fresh)
```

### Iteration 7: Understanding the Default Behavior

By default, Next.js tries to statically render everything. Let's see what that means for our product catalog:

```tsx
// src/app/products/page.tsx - Default behavior
import { getProducts, getCategories } from '@/lib/api';

export default async function ProductsPage() {
  const [products, categories] = await Promise.all([
    getProducts(),
    getCategories(),
  ]);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      
      <div className="grid grid-cols-12 gap-8">
        <aside className="col-span-3">
          <h2 className="text-xl font-semibold mb-4">Categories</h2>
          <ul className="space-y-2">
            {categories.map(cat => (
              <li key={cat.id}>{cat.name} ({cat.count})</li>
            ))}
          </ul>
        </aside>

        <main className="col-span-9">
          <div className="grid grid-cols-3 gap-6">
            {products.map(product => (
              <div key={product.id} className="border rounded-lg p-4">
                <h3 className="font-semibold">{product.name}</h3>
                <p className="text-gray-600">${product.price}</p>
              </div>
            ))}
          </div>
        </main>
      </div>
    </div>
  );
}
```

### Build This Page

Let's build the app and see what Next.js does:

```bash
npm run build
```

**Terminal Output**:
```bash
Route (app)                              Size     First Load JS
┌ ○ /                                    5.2 kB         87.3 kB
├ ○ /products                            8.4 kB         95.5 kB
└ ○ /about                               3.1 kB         85.2 kB

○  (Static)  prerendered as static content
```

**Key indicator**: The `○` symbol means the page is **statically rendered**. Next.js fetched the data and generated HTML at build time.

### Verification: Static Rendering in Production

Let's start the production server and observe the behavior:

```bash
npm run start
```

Navigate to `/products` and check the Network tab:

**Network Tab Analysis**:
- Filter: Doc (HTML document)
- Observation: Request to `/products`
- Timeline:
  - 0-5ms: Server reads pre-built HTML from disk
  - 5ms: HTML arrives (instant!)
- Pattern: No data fetching—HTML was pre-built
- **Time to First Byte (TTFB)**: 5ms ✅ (incredibly fast!)

**View Source**:
```html
<!DOCTYPE html>
<html>
<body>
  <div class="container">
    <h1>Our Products</h1>
    <!-- Full product HTML is here, generated at build time -->
    <div class="grid">
      <div>Laptop Pro - $1299</div>
      <div>Wireless Mouse - $29</div>
      <!-- All products from build time -->
    </div>
  </div>
</body>
</html>
```

**Performance Metrics**:
- **TTFB**: 5ms ✅ (no data fetching!)
- **FCP**: 50ms ✅
- **LCP**: 100ms ✅
- **TTI**: 150ms ✅

This is **incredibly fast** because the HTML is pre-built. But there's a problem...

### The Problem: Stale Data

Let's add a new product to the database and refresh the page:

1. Add product "New Laptop" to database
2. Refresh `/products` page
3. "New Laptop" doesn't appear!

**Why?** The HTML was generated at build time. The page shows the data from when you ran `npm run build`, not the current data.

### Diagnostic Analysis: Reading the Stale Data Failure

**Browser Behavior**:
- User adds new product via admin panel
- User refreshes product listing page
- New product doesn't appear
- Old products still shown

**Network Tab Analysis**:
- Request to `/products` returns instantly (5ms)
- HTML contains old data from build time
- No data fetching happens—server just serves pre-built HTML

**Server Logs**:
```bash
[No logs - no data fetching happens on request]
```

### Let's Parse This Evidence

1. **What the user experiences**:
   - Expected: See new product after adding it
   - Actual: New product doesn't appear, even after refresh

2. **What the Network tab shows**:
   - Key indicator: TTFB is 5ms—too fast to be fetching data
   - Pattern: Server is serving pre-built HTML, not generating it on request

3. **Root cause identified**: 
   Page is statically rendered at build time. Data changes after build time aren't reflected until you rebuild.

4. **Why the current approach can't solve this**:
   Static rendering is fundamentally incompatible with frequently changing data. You can't rebuild your entire site every time a product is added.

5. **What we need**:
   A way to tell Next.js to render this page dynamically on each request, so it always shows fresh data.

## Forcing Dynamic Rendering

Next.js provides several ways to opt into dynamic rendering:

### Method 1: Use Dynamic Functions

Certain functions automatically make a page dynamic:

```tsx
// src/app/products/page.tsx - Using cookies (dynamic function)
import { cookies } from 'next/headers';
import { getProducts, getCategories } from '@/lib/api';

export default async function ProductsPage() {
  // Reading cookies makes this page dynamic
  const cookieStore = cookies();
  const userPreference = cookieStore.get('view-mode');

  const [products, categories] = await Promise.all([
    getProducts(),
    getCategories(),
  ]);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      {/* Same JSX as before */}
    </div>
  );
}
```

**Dynamic functions** that force dynamic rendering:
- `cookies()` - Read request cookies
- `headers()` - Read request headers
- `searchParams` - Read URL query parameters (in page components)
- `fetch()` with `cache: 'no-store'` - Opt out of caching

### Method 2: Explicit Dynamic Configuration

You can explicitly tell Next.js to render dynamically:

```tsx
// src/app/products/page.tsx - Explicit dynamic rendering
import { getProducts, getCategories } from '@/lib/api';

// Force dynamic rendering
export const dynamic = 'force-dynamic';

export default async function ProductsPage() {
  const [products, categories] = await Promise.all([
    getProducts(),
    getCategories(),
  ]);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      {/* Same JSX as before */}
    </div>
  );
}
```

### Method 3: Uncached Fetch

Use `fetch()` with `cache: 'no-store'`:

```typescript
// src/lib/api.ts - Uncached fetch
export async function getProducts() {
  const response = await fetch('https://api.example.com/products', {
    cache: 'no-store', // Don't cache, always fetch fresh
  });
  return response.json();
}
```

### Verification: Dynamic Rendering in Production

Let's rebuild with `dynamic = 'force-dynamic'` and observe the difference:

```bash
npm run build
```

**Terminal Output**:
```bash
Route (app)                              Size     First Load JS
┌ ○ /                                    5.2 kB         87.3 kB
├ ƒ /products                            8.4 kB         95.5 kB
└ ○ /about                               3.1 kB         85.2 kB

○  (Static)   prerendered as static content
ƒ  (Dynamic)  server-rendered on demand
```

**Key indicator**: The `ƒ` symbol means the page is **dynamically rendered**. Next.js will fetch data and generate HTML on each request.

Now let's test:

1. Start production server: `npm run start`
2. Add new product to database
3. Refresh `/products` page
4. New product appears! ✅

**Network Tab Analysis**:
- Request to `/products`
- Timeline:
  - 0-500ms: Server fetching data and rendering HTML
  - 500ms: HTML arrives with fresh data
- Pattern: Data fetching happens on each request
- **TTFB**: 500ms (slower than static, but data is fresh)

**Server Logs**:
```bash
[Server] Fetching products...
[Server] Rendering page...
```

### Expected vs. Actual Improvement

**Static Rendering**:
- TTFB: 5ms ✅ (incredibly fast)
- Data freshness: Stale ❌ (only updated on rebuild)
- Scalability: Excellent ✅ (serve from CDN)
- Use case: Content that rarely changes

**Dynamic Rendering**:
- TTFB: 500ms ⚠️ (slower, but acceptable)
- Data freshness: Always fresh ✅
- Scalability: Good ⚠️ (server must render each request)
- Use case: Frequently changing or personalized content

## The Spectrum: Static, Dynamic, and Hybrid

Most real applications need a mix of both:

### Pattern 1: Static Marketing Pages, Dynamic App Pages

```typescript
// src/app/page.tsx - Static homepage
export default async function HomePage() {
  // No dynamic functions, no uncached fetches
  return (
    <div>
      <h1>Welcome to Our Store</h1>
      <p>Browse our amazing products!</p>
    </div>
  );
}
// Result: Static (○)
```

```tsx
// src/app/dashboard/page.tsx - Dynamic dashboard
import { cookies } from 'next/headers';

export default async function DashboardPage() {
  const cookieStore = cookies();
  const userId = cookieStore.get('user-id')?.value;

  // Fetch user-specific data
  const userData = await getUserData(userId);

  return (
    <div>
      <h1>Welcome back, {userData.name}!</h1>
      {/* Personalized content */}
    </div>
  );
}
// Result: Dynamic (ƒ)
```

### Pattern 2: Static Product Pages with Dynamic Cart

```tsx
// src/app/products/[id]/page.tsx - Static product details
export default async function ProductPage({
  params,
}: {
  params: { id: string };
}) {
  const product = await getProduct(params.id);

  return (
    <div>
      <h1>{product.name}</h1>
      <p>${product.price}</p>
      
      {/* Client Component for interactivity */}
      <AddToCartButton productId={product.id} />
    </div>
  );
}

// Generate static pages for all products at build time
export async function generateStaticParams() {
  const products = await getProducts();
  return products.map(product => ({
    id: product.id,
  }));
}
// Result: Static (○) with dynamic cart functionality
```

### Pattern 3: Hybrid with Partial Prerendering (Experimental)

Next.js 14+ introduces **Partial Prerendering** (PPR)—static shell with dynamic holes:

```tsx
// src/app/products/page.tsx - Partial Prerendering
import { Suspense } from 'react';

export const experimental_ppr = true;

export default async function ProductsPage() {
  // Static parts
  const categories = await getCategories();

  return (
    <div>
      {/* Static: Categories sidebar */}
      <aside>
        <h2>Categories</h2>
        <ul>
          {categories.map(cat => (
            <li key={cat.id}>{cat.name}</li>
          ))}
        </ul>
      </aside>

      {/* Dynamic: Product grid (personalized) */}
      <Suspense fallback={<ProductsLoading />}>
        <PersonalizedProducts />
      </Suspense>
    </div>
  );
}
```

**Result**: Static shell (categories) served instantly from CDN, dynamic content (personalized products) streamed in.

## How Next.js Decides: Static or Dynamic?

Next.js uses this decision tree:

1. **Does the page use dynamic functions?** (`cookies()`, `headers()`, `searchParams`)
   - Yes → Dynamic (ƒ)
   - No → Continue

2. **Does the page use uncached fetches?** (`cache: 'no-store'` or `revalidate: 0`)
   - Yes → Dynamic (ƒ)
   - No → Continue

3. **Is `dynamic = 'force-dynamic'` set?**
   - Yes → Dynamic (ƒ)
   - No → Continue

4. **Default**: Static (○)

### Debugging: Why Is My Page Dynamic?

If a page is unexpectedly dynamic, check the build output:

```bash
npm run build
```

**Terminal Output** (with explanation):
```bash
Route (app)                              Size     First Load JS
├ ƒ /products                            8.4 kB         95.5 kB

Dynamic because:
  - uses cookies() in page.tsx:12
  - uses fetch with cache: 'no-store' in api.ts:45
```

Next.js tells you exactly why a page is dynamic.

### Common Failure Modes and Their Signatures

#### Symptom: Page is dynamic when it should be static

**Build output**:
```bash
├ ƒ /products                            8.4 kB         95.5 kB
```

**Root cause**: Accidentally using dynamic functions or uncached fetches

**Solution**: Remove dynamic functions, or use `cache: 'force-cache'` for fetches

```typescript
// ❌ Bad: Makes page dynamic
export default async function Page() {
  const cookieStore = cookies(); // Dynamic function!
  // ...
}

// ✅ Good: Keep page static
export default async function Page() {
  // Don't read cookies unless you need to
  // ...
}
```

#### Symptom: Page is static when it should be dynamic

**Browser behavior**:
Stale data shown, changes don't appear

**Build output**:
```bash
├ ○ /products                            8.4 kB         95.5 kB
```

**Root cause**: Page doesn't use any dynamic functions, so Next.js assumes it's static

**Solution**: Add `export const dynamic = 'force-dynamic'` or use a dynamic function

#### Symptom: Build fails with "Page X used Y but did not export dynamic = 'force-dynamic'"

**Terminal output**:
```bash
Error: Route /products used cookies() but did not export dynamic = 'force-dynamic'
```

**Root cause**: Using dynamic functions in a page that's configured as static

**Solution**: Either remove the dynamic function or add `export const dynamic = 'force-dynamic'`

### When to Apply This Solution

**Use Static Rendering when**:
- Content rarely changes (marketing pages, docs, blog posts)
- Same content for all users (no personalization)
- Performance is critical (need sub-100ms TTFB)
- High traffic (want to serve from CDN)

**Use Dynamic Rendering when**:
- Content changes frequently (product inventory, user dashboards)
- Personalized content (recommendations, user-specific data)
- Real-time data (stock prices, live scores)
- User-specific state (shopping cart, preferences)

**Use Hybrid (Static + Dynamic) when**:
- Page has both static and dynamic parts
- Want fast initial load with personalized content
- Can use Suspense to stream dynamic parts

**Decision Framework**:

| Content Type | Frequency of Change | Personalized? | Recommendation |
|--------------|---------------------|---------------|----------------|
| Marketing pages | Rarely | No | Static |
| Blog posts | Occasionally | No | Static |
| Product catalog | Daily | No | Static + Revalidation |
| Product details | Hourly | No | Static + Revalidation |
| User dashboard | Real-time | Yes | Dynamic |
| Shopping cart | Real-time | Yes | Dynamic |
| Search results | Real-time | Maybe | Dynamic |
| Recommendations | Real-time | Yes | Dynamic (Suspense) |

### Limitation Preview

We've learned to choose between static and dynamic rendering, but there's a middle ground: **Incremental Static Regeneration (ISR)**—static pages that automatically update in the background.

What if we could get the performance of static rendering with the freshness of dynamic rendering? That's what revalidation strategies enable (next section).

## Revalidation strategies

## The Best of Both Worlds: Static Performance with Fresh Data

We've seen two extremes:

- **Static Rendering**: Fast (5ms TTFB) but stale data
- **Dynamic Rendering**: Fresh data but slower (500ms TTFB)

What if we could have both? **Incremental Static Regeneration (ISR)** gives us static performance with automatic background updates.

### The Mental Model

**Traditional Static**:
```
Build time: Generate HTML → Save to disk
Request time: Serve stale HTML forever (until next build)
```

**ISR (Incremental Static Regeneration)**:
```
Build time: Generate HTML → Save to disk
Request time: Serve cached HTML (fast!)
Background: Check if stale → Regenerate if needed → Update cache
Next request: Serve fresh HTML (still fast!)
```

### Iteration 8: Time-Based Revalidation

Let's make our product catalog update automatically every 60 seconds:

```tsx
// src/app/products/page.tsx - Time-based revalidation
import { getProducts, getCategories } from '@/lib/api';

// Revalidate this page every 60 seconds
export const revalidate = 60;

export default async function ProductsPage() {
  const [products, categories] = await Promise.all([
    getProducts(),
    getCategories(),
  ]);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      
      <div className="grid grid-cols-12 gap-8">
        <aside className="col-span-3">
          <h2 className="text-xl font-semibold mb-4">Categories</h2>
          <ul className="space-y-2">
            {categories.map(cat => (
              <li key={cat.id}>{cat.name} ({cat.count})</li>
            ))}
          </ul>
        </aside>

        <main className="col-span-9">
          <div className="grid grid-cols-3 gap-6">
            {products.map(product => (
              <div key={product.id} className="border rounded-lg p-4">
                <h3 className="font-semibold">{product.name}</h3>
                <p className="text-gray-600">${product.price}</p>
              </div>
            ))}
          </div>
        </main>
      </div>
    </div>
  );
}
```

### What Changed?

**Before** (static):
```tsx
export default async function ProductsPage() {
  // No revalidation config
  // Page is static forever
}
```

**After** (ISR):
```tsx
export const revalidate = 60; // Revalidate every 60 seconds

export default async function ProductsPage() {
  // Same code, but now it updates automatically
}
```

### How ISR Works: The Timeline

Let's trace what happens over time:

**Build time (t=0)**:
```bash
npm run build
```
- Next.js generates HTML with current product data
- HTML saved to `.next/server/app/products.html`
- Page is marked as "revalidate every 60 seconds"

**First request (t=5s)**:
- User visits `/products`
- Next.js serves pre-built HTML (5ms TTFB) ✅
- HTML shows products from build time
- No background work yet

**Second request (t=65s)** - After revalidation period:
- User visits `/products`
- Next.js serves cached HTML (5ms TTFB) ✅ (still fast!)
- **Background**: Next.js triggers revalidation
  - Fetches fresh product data
  - Renders new HTML
  - Updates cache
- User sees old data (but it's instant)

**Third request (t=70s)** - After revalidation completes:
- User visits `/products`
- Next.js serves **new** HTML (5ms TTFB) ✅
- HTML shows fresh products
- User sees updated data (and it's still instant!)

### Verification: ISR in Action

Let's test this behavior:

1. Build and start production server:

```bash
npm run build
npm run start
```

2. Visit `/products` → See products from build time (instant)
3. Add new product to database
4. Refresh `/products` immediately → Still see old products (instant, from cache)
5. Wait 60 seconds
6. Refresh `/products` → Still see old products (instant, from cache)
7. Wait a few seconds for background revalidation
8. Refresh `/products` → See new product! (instant, from updated cache)

**Network Tab Analysis** (request at t=65s):
- Request to `/products`
- Timeline:
  - 0-5ms: Server serves cached HTML
  - 5ms: HTML arrives (old data, but instant!)
- Pattern: Stale-while-revalidate—serve cached version, update in background
- **TTFB**: 5ms ✅ (static performance!)

**Server Logs** (background revalidation):
```bash
[t=65s] Request to /products
[t=65s] Serving cached HTML (age: 60s)
[t=65s] Background: Revalidation triggered
[t=65s] Background: Fetching fresh data...
[t=65.5s] Background: Rendering new HTML...
[t=65.6s] Background: Cache updated
```

### Expected vs. Actual Improvement

**Static (no revalidation)**:
- TTFB: 5ms ✅
- Data freshness: Stale until rebuild ❌
- User experience: Fast but outdated

**Dynamic (always fresh)**:
- TTFB: 500ms ⚠️
- Data freshness: Always fresh ✅
- User experience: Slow but current

**ISR (best of both)**:
- TTFB: 5ms ✅ (static performance!)
- Data freshness: Updates every 60s ✅
- User experience: Fast and reasonably fresh ✅

**Key insight**: ISR gives you static performance with automatic updates. Users always get instant responses, and data stays reasonably fresh.

## Revalidation Strategies

Next.js provides multiple ways to revalidate cached pages:

### Strategy 1: Time-Based Revalidation

Revalidate after a fixed time period:

```tsx
// src/app/products/page.tsx - Revalidate every 60 seconds
export const revalidate = 60;

export default async function ProductsPage() {
  // Page regenerates in background every 60 seconds
}
```

**Use when**:
- Data changes predictably (e.g., every hour)
- Acceptable to show slightly stale data
- High traffic (want to minimize server load)

**Examples**:
- Product catalog (revalidate every 5 minutes)
- Blog posts (revalidate every hour)
- Stock prices (revalidate every 30 seconds)

### Strategy 2: On-Demand Revalidation

Revalidate immediately when data changes:

```typescript
// src/app/api/revalidate/route.ts - API route for on-demand revalidation
import { revalidatePath } from 'next/cache';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const { path, secret } = await request.json();

  // Verify secret to prevent unauthorized revalidation
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ message: 'Invalid secret' }, { status: 401 });
  }

  try {
    // Revalidate the specified path
    revalidatePath(path);
    return NextResponse.json({ revalidated: true, now: Date.now() });
  } catch (err) {
    return NextResponse.json({ message: 'Error revalidating' }, { status: 500 });
  }
}
```

```typescript
// src/lib/admin.ts - Trigger revalidation after data change
export async function addProduct(product: Product) {
  // Add product to database
  await db.product.create({ data: product });

  // Trigger revalidation
  await fetch('https://yoursite.com/api/revalidate', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      path: '/products',
      secret: process.env.REVALIDATION_SECRET,
    }),
  });
}
```

**Use when**:
- Data changes unpredictably
- Need immediate updates (e.g., after admin action)
- Can trigger revalidation from your backend

**Examples**:
- Product added/updated by admin → Revalidate `/products`
- Blog post published → Revalidate `/blog`
- Inventory updated → Revalidate `/products/[id]`

### Strategy 3: Tag-Based Revalidation

Revalidate multiple related pages at once:

```typescript
// src/lib/api.ts - Tag fetches for revalidation
export async function getProducts() {
  const response = await fetch('https://api.example.com/products', {
    next: {
      tags: ['products'], // Tag this fetch
      revalidate: 60,
    },
  });
  return response.json();
}

export async function getProduct(id: string) {
  const response = await fetch(`https://api.example.com/products/${id}`, {
    next: {
      tags: ['products', `product-${id}`], // Multiple tags
      revalidate: 60,
    },
  });
  return response.json();
}
```

```typescript
// src/app/api/revalidate/route.ts - Revalidate by tag
import { revalidateTag } from 'next/cache';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const { tag, secret } = await request.json();

  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ message: 'Invalid secret' }, { status: 401 });
  }

  try {
    // Revalidate all fetches with this tag
    revalidateTag(tag);
    return NextResponse.json({ revalidated: true, now: Date.now() });
  } catch (err) {
    return NextResponse.json({ message: 'Error revalidating' }, { status: 500 });
  }
}
```

```typescript
// src/lib/admin.ts - Revalidate all product pages
export async function updateProduct(id: string, updates: Partial<Product>) {
  await db.product.update({ where: { id }, data: updates });

  // Revalidate all pages that use product data
  await fetch('https://yoursite.com/api/revalidate', {
    method: 'POST',
    body: JSON.stringify({
      tag: 'products', // Revalidates /products, /products/[id], etc.
      secret: process.env.REVALIDATION_SECRET,
    }),
  });
}
```

**Use when**:
- Multiple pages depend on the same data
- Want to revalidate related pages together
- Complex data dependencies

**Examples**:
- Update product → Revalidate product list, product detail, category pages
- Update user profile → Revalidate dashboard, settings, profile pages

### Strategy 4: Fetch-Level Revalidation

Different revalidation times for different data sources:

```typescript
// src/lib/api.ts - Per-fetch revalidation
export async function getProducts() {
  const response = await fetch('https://api.example.com/products', {
    next: { revalidate: 300 }, // 5 minutes
  });
  return response.json();
}

export async function getCategories() {
  const response = await fetch('https://api.example.com/categories', {
    next: { revalidate: 3600 }, // 1 hour (changes rarely)
  });
  return response.json();
}

export async function getFeaturedProducts() {
  const response = await fetch('https://api.example.com/featured', {
    next: { revalidate: 60 }, // 1 minute (changes frequently)
  });
  return response.json();
}
```

```tsx
// src/app/products/page.tsx - Mixed revalidation times
export default async function ProductsPage() {
  // Each fetch has its own revalidation time
  const [products, categories, featured] = await Promise.all([
    getProducts(),      // Revalidates every 5 minutes
    getCategories(),    // Revalidates every hour
    getFeaturedProducts(), // Revalidates every minute
  ]);

  return (
    <div>
      {/* Categories update hourly */}
      <CategorySidebar categories={categories} />
      
      {/* Products update every 5 minutes */}
      <ProductGrid products={products} />
      
      {/* Featured updates every minute */}
      <FeaturedBanner products={featured} />
    </div>
  );
}
```

**Use when**:
- Different data sources have different update frequencies
- Want fine-grained control over caching
- Optimizing for both performance and freshness

### Strategy 5: No Revalidation (Cache Forever)

Some data never changes:

```typescript
// src/lib/api.ts - Cache forever
export async function getCountries() {
  const response = await fetch('https://api.example.com/countries', {
    next: { revalidate: false }, // Cache forever
  });
  return response.json();
}
```

**Use when**:
- Data is truly static (country list, currency codes)
- Content is versioned (e.g., `/blog/post-v1`, `/blog/post-v2`)

## Real-World Revalidation Patterns

### Pattern 1: E-commerce Product Catalog

```tsx
// src/app/products/page.tsx - Product listing
export const revalidate = 300; // 5 minutes

export default async function ProductsPage() {
  const products = await getProducts();
  return <ProductGrid products={products} />;
}
```

```tsx
// src/app/products/[id]/page.tsx - Product details
export default async function ProductPage({
  params,
}: {
  params: { id: string };
}) {
  const product = await getProduct(params.id);
  return <ProductDetails product={product} />;
}

// Generate static pages for all products at build time
export async function generateStaticParams() {
  const products = await getProducts();
  return products.map(product => ({ id: product.id }));
}

// Revalidate individual product pages every 5 minutes
export const revalidate = 300;
```

```typescript
// src/lib/admin.ts - Admin updates trigger revalidation
export async function updateProductInventory(id: string, quantity: number) {
  await db.product.update({
    where: { id },
    data: { inventory: quantity },
  });

  // Immediately revalidate this product page
  await fetch('https://yoursite.com/api/revalidate', {
    method: 'POST',
    body: JSON.stringify({
      path: `/products/${id}`,
      secret: process.env.REVALIDATION_SECRET,
    }),
  });
}
```

**Result**:
- Product pages are static (fast!)
- Update every 5 minutes automatically
- Admin changes trigger immediate updates
- Best of all worlds: fast, fresh, and responsive to changes

### Pattern 2: Blog with Instant Publishing

```tsx
// src/app/blog/page.tsx - Blog listing
export const revalidate = 3600; // 1 hour

export default async function BlogPage() {
  const posts = await getPosts();
  return <PostList posts={posts} />;
}
```

```tsx
// src/app/blog/[slug]/page.tsx - Blog post
export default async function PostPage({
  params,
}: {
  params: { slug: string };
}) {
  const post = await getPost(params.slug);
  return <PostContent post={post} />;
}

export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map(post => ({ slug: post.slug }));
}

// Posts rarely change after publishing
export const revalidate = false; // Cache forever
```

```typescript
// src/lib/cms.ts - CMS webhook triggers revalidation
export async function handlePublishWebhook(postSlug: string) {
  // Revalidate the new post and the blog listing
  await Promise.all([
    fetch('https://yoursite.com/api/revalidate', {
      method: 'POST',
      body: JSON.stringify({
        path: `/blog/${postSlug}`,
        secret: process.env.REVALIDATION_SECRET,
      }),
    }),
    fetch('https://yoursite.com/api/revalidate', {
      method: 'POST',
      body: JSON.stringify({
        path: '/blog',
        secret: process.env.REVALIDATION_SECRET,
      }),
    }),
  ]);
}
```

**Result**:
- Blog posts are static (instant loading)
- Cached forever (they don't change)
- Publishing triggers immediate revalidation
- Blog listing updates hourly

### Pattern 3: Dashboard with Mixed Freshness

```tsx
// src/app/dashboard/page.tsx - Dashboard with mixed data
import { Suspense } from 'react';

export default async function DashboardPage() {
  // Fast, cached data
  const stats = await getStats(); // Revalidates every 5 minutes

  return (
    <div>
      {/* Static stats */}
      <StatsCards stats={stats} />

      {/* Real-time data (dynamic) */}
      <Suspense fallback={<ActivityLoading />}>
        <RecentActivity /> {/* Always fresh, no caching */}
      </Suspense>

      {/* Cached data */}
      <Suspense fallback={<ChartLoading />}>
        <RevenueChart /> {/* Revalidates every hour */}
      </Suspense>
    </div>
  );
}
```

```typescript
// src/lib/api.ts - Different revalidation for different data
export async function getStats() {
  const response = await fetch('https://api.example.com/stats', {
    next: { revalidate: 300 }, // 5 minutes
  });
  return response.json();
}

export async function getRecentActivity() {
  const response = await fetch('https://api.example.com/activity', {
    cache: 'no-store', // Always fresh, no caching
  });
  return response.json();
}

export async function getRevenueData() {
  const response = await fetch('https://api.example.com/revenue', {
    next: { revalidate: 3600 }, // 1 hour
  });
  return response.json();
}
```

**Result**:
- Stats update every 5 minutes (good enough for most users)
- Recent activity is always fresh (real-time)
- Revenue chart updates hourly (expensive query, cached longer)
- Page loads fast with progressive enhancement

## Common Failure Modes and Their Signatures

### Symptom: Revalidation not working (data stays stale)

**Browser behavior**:
Data doesn't update even after revalidation period

**Server logs**:
```bash
[No revalidation logs - revalidation not triggering]
```

**Root cause**: Page is using `dynamic = 'force-dynamic'` or dynamic functions, which disables ISR

**Solution**: Remove dynamic functions or use fetch-level revalidation instead

```tsx
// ❌ Bad: Dynamic page can't use ISR
export const dynamic = 'force-dynamic';
export const revalidate = 60; // This is ignored!

export default async function Page() {
  const data = await getData();
  return <div>{data}</div>;
}

// ✅ Good: Static page with ISR
export const revalidate = 60;

export default async function Page() {
  const data = await getData();
  return <div>{data}</div>;
}
```

### Symptom: On-demand revalidation returns 401 Unauthorized

**API response**:
```json
{ "message": "Invalid secret" }
```

**Root cause**: Revalidation secret doesn't match

**Solution**: Check environment variable is set correctly

```bash
# .env.local
REVALIDATION_SECRET=your-secret-key-here
```

### Symptom: Revalidation triggers too frequently (high server load)

**Server logs**:
```bash
[10:00:00] Revalidating /products
[10:00:05] Revalidating /products
[10:00:10] Revalidating /products
[10:00:15] Revalidating /products
```

**Root cause**: Revalidation period is too short for high-traffic pages

**Solution**: Increase revalidation time or use on-demand revalidation

```tsx
// ❌ Bad: Revalidates every 5 seconds (too frequent!)
export const revalidate = 5;

// ✅ Good: Revalidates every 5 minutes
export const revalidate = 300;

// ✅ Better: Use on-demand revalidation for immediate updates
export const revalidate = 3600; // 1 hour baseline
// Trigger revalidation manually when data changes
```

### Symptom: Some users see old data, others see new data

**Browser behavior**:
Inconsistent data across users

**Root cause**: CDN caching—different edge locations have different cache states

**Solution**: This is expected behavior with ISR. Use shorter revalidation times or on-demand revalidation for critical updates.

## The Complete Data Fetching Journey

Let's trace our product catalog through all the iterations:

| Iteration | Approach | TTFB | Data Freshness | Complexity | Use Case |
|-----------|----------|------|----------------|------------|----------|
| 0 | Client-side fetch | 2200ms | Real-time | High | ❌ Don't use |
| 1 | Server Component (sequential) | 2250ms | Real-time | Low | Rarely needed |
| 2 | Server Component (parallel) | 873ms | Real-time | Low | Dynamic pages |
| 3 | Client Component + React Query | 100ms (cached) | Real-time | Medium | Interactive features |
| 4 | Streaming with Suspense | 600ms (fast parts) | Real-time | Medium | Mixed fast/slow data |
| 5 | Static rendering | 5ms | Stale | Low | Marketing pages |
| 6 | Dynamic rendering | 873ms | Real-time | Low | User dashboards |
| 7 | ISR (time-based) | 5ms | Updates every 60s | Low | Product catalogs |
| 8 | ISR (on-demand) | 5ms | Updates on change | Medium | Admin-managed content |

### Decision Framework: Which Approach When?

**Use Server Components when**:
- Initial page load data
- SEO is important
- Data is not user-specific
- Can tolerate some staleness

**Use Client Components + React Query when**:
- Interactive features (search, filters)
- Real-time updates
- User-triggered actions
- Need caching and optimistic updates

**Use Streaming when**:
- Page has mix of fast and slow data
- Want to show partial content quickly
- User experience is critical

**Use Static Rendering when**:
- Content rarely changes
- Same for all users
- Performance is critical
- High traffic

**Use Dynamic Rendering when**:
- Content changes frequently
- Personalized for each user
- Real-time data required
- Low to medium traffic

**Use ISR when**:
- Content changes periodically
- Can tolerate slight staleness
- Want static performance
- High traffic

### The Professional React Developer's Mental Model

When approaching data fetching in Next.js, ask these questions:

1. **Who needs this data?**
   - All users → Consider static/ISR
   - Specific user → Consider dynamic

2. **How often does it change?**
   - Never → Static
   - Rarely → ISR (long revalidation)
   - Hourly → ISR (short revalidation)
   - Real-time → Dynamic or Client Component

3. **How fast must it load?**
   - Critical (< 100ms) → Static or ISR
   - Important (< 500ms) → Server Component or ISR
   - Acceptable (< 2s) → Dynamic or Streaming

4. **Is it interactive?**
   - Yes → Client Component
   - No → Server Component

5. **Does it need to be fresh?**
   - Always → Dynamic or Client Component
   - Eventually → ISR
   - Doesn't matter → Static

**The golden rule**: Start with Server Components and ISR. Only reach for dynamic rendering or client-side fetching when you have a specific reason.
