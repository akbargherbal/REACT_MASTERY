# Chapter 16: App Router Fundamentals

## File-based routing

## File-based routing

Before we dive into Server Components and the architectural revolution they represent, we need to understand how Next.js organizes your application. The App Router uses your file system as your routing configuration—no separate routing file, no manual route registration, just files and folders.

This isn't just a convenience feature. File-based routing is the foundation that makes Server Components, streaming, and parallel data fetching possible. Understanding this structure is essential before we can explore what makes Next.js different from client-only React.

### The Problem: Manual Route Configuration Doesn't Scale

In Chapter 14, we built a documentation site with React Router. Every route required explicit configuration:

```tsx
// React Router approach - manual configuration
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/products" element={<ProductList />} />
        <Route path="/products/:id" element={<ProductDetail />} />
        <Route path="/cart" element={<Cart />} />
        <Route path="/checkout" element={<Checkout />} />
        {/* Add 50 more routes... */}
      </Routes>
    </BrowserRouter>
  );
}
```

This works, but it creates friction:

1. **Disconnected structure**: Routes are defined in one place, components in another
2. **Manual maintenance**: Every new page requires updating the routing configuration
3. **No conventions**: Developers must decide how to organize files
4. **Difficult refactoring**: Moving a route means updating multiple files

For a small app, this is manageable. For a large application with hundreds of routes, it becomes a maintenance burden.

### Next.js Solution: Your File System IS Your Router

Next.js eliminates the routing configuration file entirely. The structure of your `app` directory defines your routes automatically.

**Project Structure**:

```bash
app/
├── page.tsx                    # Route: /
├── products/
│   ├── page.tsx               # Route: /products
│   └── [id]/
│       └── page.tsx           # Route: /products/:id
├── cart/
│   └── page.tsx               # Route: /cart
└── checkout/
    └── page.tsx               # Route: /checkout
```

Each `page.tsx` file automatically becomes a route. The folder structure defines the URL structure. No configuration needed.

### Reference Implementation: E-commerce Product Catalog

We'll build a product catalog that demonstrates every routing pattern you'll need in production. This will be our anchor example throughout this chapter.

**Initial Structure**:

```bash
app/
├── page.tsx                    # Home page
├── products/
│   ├── page.tsx               # Product listing
│   └── [id]/
│       └── page.tsx           # Product detail
└── layout.tsx                 # Root layout (we'll explore this in 16.3)
```

Let's start with the home page:

```tsx
// app/page.tsx
export default function HomePage() {
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-4xl font-bold mb-4">Welcome to TechStore</h1>
      <p className="text-lg text-gray-600 mb-8">
        Discover the latest in technology and gadgets
      </p>
      <a 
        href="/products" 
        className="bg-blue-600 text-white px-6 py-3 rounded-lg hover:bg-blue-700"
      >
        Browse Products
      </a>
    </div>
  );
}
```

Notice: No routing imports, no route configuration. This file exists at `app/page.tsx`, so it's automatically the home page.

Now the product listing:

```tsx
// app/products/page.tsx
export default function ProductsPage() {
  // Hardcoded for now - we'll fetch real data in Chapter 18
  const products = [
    { id: '1', name: 'Wireless Headphones', price: 99.99 },
    { id: '2', name: 'Smart Watch', price: 299.99 },
    { id: '3', name: 'Laptop Stand', price: 49.99 },
  ];

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-6">Products</h1>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {products.map((product) => (
          <div key={product.id} className="border rounded-lg p-4">
            <h2 className="text-xl font-semibold mb-2">{product.name}</h2>
            <p className="text-gray-600 mb-4">${product.price}</p>
            <a 
              href={`/products/${product.id}`}
              className="text-blue-600 hover:underline"
            >
              View Details
            </a>
          </div>
        ))}
      </div>
    </div>
  );
}
```

This file is at `app/products/page.tsx`, so it's automatically available at `/products`.

### Dynamic Routes: The `[param]` Convention

The product detail page needs to handle any product ID. In React Router, we used `:id`. In Next.js, we use `[id]`:

```tsx
// app/products/[id]/page.tsx
interface ProductPageProps {
  params: {
    id: string;
  };
}

export default function ProductPage({ params }: ProductPageProps) {
  // In a real app, we'd fetch the product by ID
  // For now, just display the ID
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">Product Details</h1>
      <p className="text-gray-600">Product ID: {params.id}</p>
      <a href="/products" className="text-blue-600 hover:underline mt-4 block">
        ← Back to Products
      </a>
    </div>
  );
}
```

**Key observations**:

1. **Folder name defines the parameter**: `[id]` becomes `params.id`
2. **Type-safe by default**: TypeScript knows the shape of `params`
3. **No route configuration**: The file structure is the configuration

**Run the app**:

```bash
npm run dev
```

**Browser behavior**:
- Navigate to `http://localhost:3000` → Home page renders
- Click "Browse Products" → `/products` page renders
- Click "View Details" on any product → `/products/1` (or 2, 3) renders
- URL changes, but no full page reload (Next.js handles client-side navigation)

**Browser Console**:
```
(No errors - everything works)
```

This is our baseline. Now let's explore what happens when we try to use client-side navigation patterns incorrectly.

### The Failure: Using `<a>` Tags Causes Full Page Reloads

Our current implementation works, but it's not optimal. Let's verify this by watching the Network tab.

**Diagnostic Analysis: Reading the Network Evidence**

**Browser DevTools - Network Tab**:
1. Open DevTools (F12)
2. Go to Network tab
3. Click "Browse Products" link
4. Observe: Full page reload
   - New document request to `/products`
   - All JavaScript bundles re-downloaded
   - React re-initializes from scratch

**Performance Impact**:
- **Before navigation**: Page fully loaded, React hydrated
- **After navigation**: 200-300ms delay while new page loads
- **User experience**: Brief white flash, scroll position lost

**Let's parse this evidence**:

1. **What the user experiences**: 
   - Expected: Instant navigation like a single-page app
   - Actual: Brief loading delay, feels like a traditional website

2. **What the Network tab reveals**:
   - Key indicator: `document` type request for `/products`
   - This means: Browser is loading an entirely new HTML page
   - Evidence: All assets (JS, CSS) are re-requested

3. **Root cause identified**: Using `<a href>` triggers browser's default navigation behavior

4. **Why the current approach can't solve this**: HTML `<a>` tags always cause full page navigation

5. **What we need**: Client-side navigation that updates the URL without reloading the page

### Iteration 1: Client-Side Navigation with `<Link>`

Next.js provides a `Link` component that handles client-side navigation:

```tsx
// app/page.tsx - Updated with Link
import Link from 'next/link';

export default function HomePage() {
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-4xl font-bold mb-4">Welcome to TechStore</h1>
      <p className="text-lg text-gray-600 mb-8">
        Discover the latest in technology and gadgets
      </p>
      <Link 
        href="/products" 
        className="bg-blue-600 text-white px-6 py-3 rounded-lg hover:bg-blue-700"
      >
        Browse Products
      </Link>
    </div>
  );
}
```

```tsx
// app/products/page.tsx - Updated with Link
import Link from 'next/link';

export default function ProductsPage() {
  const products = [
    { id: '1', name: 'Wireless Headphones', price: 99.99 },
    { id: '2', name: 'Smart Watch', price: 299.99 },
    { id: '3', name: 'Laptop Stand', price: 49.99 },
  ];

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-6">Products</h1>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {products.map((product) => (
          <div key={product.id} className="border rounded-lg p-4">
            <h2 className="text-xl font-semibold mb-2">{product.name}</h2>
            <p className="text-gray-600 mb-4">${product.price}</p>
            <Link 
              href={`/products/${product.id}`}
              className="text-blue-600 hover:underline"
            >
              View Details
            </Link>
          </div>
        ))}
      </div>
    </div>
  );
}
```

```tsx
// app/products/[id]/page.tsx - Updated with Link
import Link from 'next/link';

interface ProductPageProps {
  params: {
    id: string;
  };
}

export default function ProductPage({ params }: ProductPageProps) {
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">Product Details</h1>
      <p className="text-gray-600">Product ID: {params.id}</p>
      <Link href="/products" className="text-blue-600 hover:underline mt-4 block">
        ← Back to Products
      </Link>
    </div>
  );
}
```

**Verification**: Rerun the navigation flow with Network tab open.

**Browser DevTools - Network Tab** (After fix):
- Click "Browse Products"
- Observe: No document reload
- Only: Small JSON payload fetched (React Server Component payload)
- Size: ~2KB instead of ~200KB full page reload

**Performance Metrics**:
- **Before** (with `<a>`):
  - Navigation time: 200-300ms
  - Data transferred: ~200KB (full page)
  - User experience: Brief white flash
  
- **After** (with `<Link>`):
  - Navigation time: 50-100ms
  - Data transferred: ~2KB (component data only)
  - User experience: Instant, smooth transition

**Expected vs. Actual improvement**: Navigation is now 3-4x faster and feels instant. No white flash, no scroll position loss.

### Route Organization Patterns

Now that we understand basic routing, let's explore how to organize complex applications.

#### Pattern 1: Route Groups (Organization Without URL Impact)

Sometimes you want to organize files without affecting URLs. Use parentheses:

```bash
app/
├── (marketing)/
│   ├── page.tsx              # Route: / (not /marketing)
│   ├── about/
│   │   └── page.tsx          # Route: /about (not /marketing/about)
│   └── contact/
│       └── page.tsx          # Route: /contact
└── (shop)/
    ├── products/
    │   └── page.tsx          # Route: /products
    └── cart/
        └── page.tsx          # Route: /cart
```

The `(marketing)` and `(shop)` folders are purely organizational—they don't appear in URLs. This is useful for:
- Grouping related routes
- Applying different layouts to different sections (we'll see this in 16.3)
- Team-based code organization

#### Pattern 2: Catch-All Routes

For documentation or blog systems, you might need to match multiple path segments:

```bash
app/
└── docs/
    └── [...slug]/
        └── page.tsx          # Matches /docs/a, /docs/a/b, /docs/a/b/c
```

```tsx
// app/docs/[...slug]/page.tsx
interface DocsPageProps {
  params: {
    slug: string[];  // Array of path segments
  };
}

export default function DocsPage({ params }: DocsPageProps) {
  // /docs/getting-started → params.slug = ['getting-started']
  // /docs/api/authentication → params.slug = ['api', 'authentication']
  
  const path = params.slug.join('/');
  
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">Documentation</h1>
      <p className="text-gray-600">Path: {path}</p>
    </div>
  );
}
```

#### Pattern 3: Optional Catch-All Routes

To match both the base route AND nested routes:

```bash
app/
└── docs/
    └── [[...slug]]/
        └── page.tsx          # Matches /docs AND /docs/a AND /docs/a/b
```

```tsx
// app/docs/[[...slug]]/page.tsx
interface DocsPageProps {
  params: {
    slug?: string[];  // Optional - might be undefined for /docs
  };
}

export default function DocsPage({ params }: DocsPageProps) {
  const path = params.slug ? params.slug.join('/') : 'index';
  
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">Documentation</h1>
      <p className="text-gray-600">Path: {path}</p>
    </div>
  );
}
```

### Special Files in the App Router

Beyond `page.tsx`, Next.js recognizes several special files:

| File | Purpose | Example Use |
|------|---------|-------------|
| `page.tsx` | Route UI | The actual page content |
| `layout.tsx` | Shared UI wrapper | Navigation, footer (covered in 16.3) |
| `loading.tsx` | Loading UI | Skeleton screens (covered in 16.4) |
| `error.tsx` | Error UI | Error boundaries (covered in 16.4) |
| `not-found.tsx` | 404 UI | Custom 404 page |
| `route.ts` | API endpoint | REST API routes (Chapter 19) |

We'll explore layouts, loading, and error states in the next sections. For now, understand that these files have special meaning in the App Router.

### When to Apply: File-Based Routing Decision Framework

**Use file-based routing when**:
- Building a Next.js application (it's the only option)
- You want automatic code splitting per route
- You need server-side rendering or static generation
- You value convention over configuration

**Characteristics**:
- **Setup complexity**: Minimal - just create files
- **Maintenance burden**: Low - structure is self-documenting
- **Performance impact**: Positive - automatic optimizations
- **Learning curve**: Gentle - intuitive folder structure

**Limitation preview**: File-based routing is powerful, but we haven't addressed how to share UI between routes (navigation, footers) or handle loading/error states. That's what layouts and special files solve, which we'll explore next.

## Server Components vs. Client Components

## Server Components vs. Client Components

Now we reach the architectural revolution that makes Next.js fundamentally different from client-only React. This is not a minor feature—it's a complete rethinking of how React applications work.

In Chapters 1-15, every React component we wrote ran in the browser. The server sent HTML, JavaScript downloaded, React hydrated, and then your components executed client-side. This is how React has always worked.

Next.js 13+ introduces **React Server Components**: components that run on the server, render to a special format, and never send their code to the browser. This isn't server-side rendering (SSR) in the traditional sense—it's a new paradigm.

### The Problem: Everything Runs in the Browser

Let's expose the issue by building a product detail page that fetches data:

```tsx
// app/products/[id]/page.tsx - Client-only approach (WRONG in Next.js)
'use client';  // This makes it a Client Component

import { useState, useEffect } from 'react';

interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  category: string;
}

export default function ProductPage({ params }: { params: { id: string } }) {
  const [product, setProduct] = useState<Product | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/products/${params.id}`)
      .then(res => res.json())
      .then(data => {
        setProduct(data);
        setIsLoading(false);
      });
  }, [params.id]);

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (!product) {
    return <div>Product not found</div>;
  }

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
      <p className="text-gray-600 mb-4">{product.description}</p>
      <p className="text-2xl font-bold mb-4">${product.price}</p>
      <p className="text-sm text-gray-500">Category: {product.category}</p>
    </div>
  );
}
```

This is the pattern we learned in Chapter 4. It works, but let's examine what actually happens.

**Diagnostic Analysis: Reading the Client-Side Waterfall**

**Browser DevTools - Network Tab**:
1. Initial page load: `/products/1`
   - HTML arrives: ~2KB
   - JavaScript bundles: ~180KB (React + our component code)
   - Total time: ~400ms

2. After JavaScript executes:
   - API request: `/api/products/1`
   - Response: ~1KB JSON
   - Time: ~150ms

3. Total time to interactive: ~550ms

**Browser Console**:
```
(No errors, but let's check what loaded)
```

**React DevTools - Components Tab**:
- `ProductPage` component visible
- State: `{ product: null, isLoading: true }` initially
- After fetch: `{ product: {...}, isLoading: false }`

**View Source** (Right-click → View Page Source):
```html
<div>Loading...</div>
```

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Product information immediately
   - Actual: "Loading..." spinner, then content appears

2. **What the Network tab reveals**:
   - Key indicator: Sequential waterfall (HTML → JS → API)
   - JavaScript must download and execute before data fetching begins
   - Total blocking time: ~400ms before data fetch even starts

3. **What View Source shows**:
   - HTML contains only loading state
   - No product information in initial HTML
   - Search engines see "Loading..." (SEO problem)

4. **Root cause identified**: Data fetching happens client-side, after JavaScript loads

5. **Why the current approach can't solve this**: Client Components must wait for JavaScript before they can do anything

6. **What we need**: A way to fetch data on the server and send rendered HTML

### Understanding the Two Component Types

Next.js introduces a fundamental distinction:

**Server Components** (default):
- Run on the server during the request
- Can directly access databases, file systems, environment variables
- Never send their code to the browser
- Cannot use hooks like `useState`, `useEffect`
- Cannot handle browser events

**Client Components** (opt-in with `'use client'`):
- Run in the browser (and during SSR)
- Can use all React hooks
- Can handle user interactions
- Their code is sent to the browser
- Cannot directly access server-only resources

### Iteration 2: Server Component Data Fetching

Let's rewrite the product page as a Server Component:

```tsx
// app/products/[id]/page.tsx - Server Component approach
// NO 'use client' directive - this is a Server Component by default

interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  category: string;
}

// Simulate database fetch - in production, this would be a real DB query
async function getProduct(id: string): Promise<Product> {
  // This runs on the server, so we can access environment variables,
  // databases, file systems, etc.
  const res = await fetch(`https://api.example.com/products/${id}`, {
    // Server-side fetch can use secret API keys
    headers: {
      'Authorization': `Bearer ${process.env.API_SECRET_KEY}`
    }
  });
  
  if (!res.ok) {
    throw new Error('Failed to fetch product');
  }
  
  return res.json();
}

// Server Components can be async!
export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  // Fetch data directly in the component
  const product = await getProduct(params.id);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
      <p className="text-gray-600 mb-4">{product.description}</p>
      <p className="text-2xl font-bold mb-4">${product.price}</p>
      <p className="text-sm text-gray-500">Category: {product.category}</p>
    </div>
  );
}
```

**Key differences**:

1. **No `'use client'` directive**: Server Component by default
2. **`async` function**: Server Components can be async
3. **Direct data fetching**: No `useEffect`, no loading state
4. **Server-only code**: Can use `process.env` directly

**Verification**: Run the app and check the Network tab.

**Browser DevTools - Network Tab** (After fix):
1. Initial page load: `/products/1`
   - HTML arrives: ~3KB (includes rendered product data)
   - JavaScript bundles: ~120KB (60KB smaller - no fetch logic)
   - Total time: ~200ms

2. No API request needed - data already in HTML

**View Source** (Right-click → View Page Source):
```html
<div class="container mx-auto px-4 py-8">
  <h1 class="text-3xl font-bold mb-4">Wireless Headphones</h1>
  <p class="text-gray-600 mb-4">Premium noise-cancelling headphones...</p>
  <p class="text-2xl font-bold mb-4">$99.99</p>
  <p class="text-sm text-gray-500">Category: Audio</p>
</div>
```

**Performance Metrics**:
- **Before** (Client Component):
  - Time to interactive: ~550ms
  - JavaScript bundle: ~180KB
  - SEO: Poor (only "Loading..." in HTML)
  - API requests: 1 (client-side)

- **After** (Server Component):
  - Time to interactive: ~200ms (63% faster)
  - JavaScript bundle: ~120KB (33% smaller)
  - SEO: Excellent (full content in HTML)
  - API requests: 0 (server-side only)

**Expected vs. Actual improvement**: Page loads 2.75x faster, bundle is 33% smaller, and search engines see full content immediately.

### The Failure: Trying to Add Interactivity to Server Components

Our product page now loads fast, but it's completely static. Let's try to add a "Add to Cart" button:

```tsx
// app/products/[id]/page.tsx - Attempting to add interactivity (WILL FAIL)
async function getProduct(id: string): Promise<Product> {
  // ... same as before
}

export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  const product = await getProduct(params.id);

  // Try to add click handler
  const handleAddToCart = () => {
    console.log('Adding to cart:', product.id);
  };

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
      <p className="text-gray-600 mb-4">{product.description}</p>
      <p className="text-2xl font-bold mb-4">${product.price}</p>
      <p className="text-sm text-gray-500">Category: {product.category}</p>
      
      <button 
        onClick={handleAddToCart}
        className="bg-blue-600 text-white px-6 py-3 rounded-lg"
      >
        Add to Cart
      </button>
    </div>
  );
}
```

**Terminal Output**:
```bash
Error: Event handlers cannot be passed to Client Component props.
  <button onClick={function} ...>
                   ^^^^^^^^^^
If you need interactivity, consider converting part of this to a Client Component.
```

**Let's parse this evidence**:

1. **What the developer experiences**:
   - Expected: Button with click handler
   - Actual: Build error, app won't compile

2. **What the error reveals**:
   - Key indicator: "Event handlers cannot be passed to Client Component props"
   - This means: Server Components can't handle browser events
   - The button is rendered, but the `onClick` can't work

3. **Root cause identified**: Server Components run on the server, where there are no click events

4. **Why the current approach can't solve this**: Server Components fundamentally can't handle interactivity

5. **What we need**: A way to mix Server Components (for data) with Client Components (for interactivity)

### Iteration 3: Hybrid Architecture - Server + Client Components

The solution is to use Server Components for data fetching and Client Components for interactivity. Let's split our component:

```tsx
// app/products/[id]/page.tsx - Server Component (data fetching)
import AddToCartButton from './AddToCartButton';

interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  category: string;
}

async function getProduct(id: string): Promise<Product> {
  const res = await fetch(`https://api.example.com/products/${id}`, {
    headers: {
      'Authorization': `Bearer ${process.env.API_SECRET_KEY}`
    }
  });
  
  if (!res.ok) {
    throw new Error('Failed to fetch product');
  }
  
  return res.json();
}

export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  // Server Component fetches data
  const product = await getProduct(params.id);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
      <p className="text-gray-600 mb-4">{product.description}</p>
      <p className="text-2xl font-bold mb-4">${product.price}</p>
      <p className="text-sm text-gray-500 mb-6">Category: {product.category}</p>
      
      {/* Client Component handles interactivity */}
      <AddToCartButton productId={product.id} productName={product.name} />
    </div>
  );
}
```

```tsx
// app/products/[id]/AddToCartButton.tsx - Client Component (interactivity)
'use client';  // This makes it a Client Component

import { useState } from 'react';

interface AddToCartButtonProps {
  productId: string;
  productName: string;
}

export default function AddToCartButton({ 
  productId, 
  productName 
}: AddToCartButtonProps) {
  const [isAdding, setIsAdding] = useState(false);
  const [isAdded, setIsAdded] = useState(false);

  const handleAddToCart = async () => {
    setIsAdding(true);
    
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 500));
    
    console.log('Added to cart:', productId);
    setIsAdding(false);
    setIsAdded(true);
    
    // Reset after 2 seconds
    setTimeout(() => setIsAdded(false), 2000);
  };

  return (
    <button 
      onClick={handleAddToCart}
      disabled={isAdding}
      className={`px-6 py-3 rounded-lg font-semibold transition-colors ${
        isAdded 
          ? 'bg-green-600 text-white' 
          : 'bg-blue-600 text-white hover:bg-blue-700'
      } disabled:opacity-50`}
    >
      {isAdding ? 'Adding...' : isAdded ? 'Added!' : 'Add to Cart'}
    </button>
  );
}
```

**Verification**: Run the app and test the button.

**Browser behavior**:
- Page loads with full product data immediately (Server Component)
- Button is interactive (Client Component)
- Click "Add to Cart" → Button shows "Adding..." → "Added!" → back to "Add to Cart"
- Console logs: "Added to cart: 1"

**Browser DevTools - Network Tab**:
- Initial load: HTML includes product data
- JavaScript bundle: Only includes `AddToCartButton` code, not data fetching logic
- Bundle size: ~125KB (still smaller than full client-side approach)

**React DevTools - Components Tab**:
- `ProductPage` not visible (it's a Server Component)
- `AddToCartButton` visible with state: `{ isAdding: false, isAdded: false }`

**Expected vs. Actual improvement**: We now have the best of both worlds—fast initial load with server-rendered data, plus full interactivity where needed.

### The Component Boundary Rules

Understanding where to place the `'use client'` directive is crucial:

**Rule 1: Client boundary is at the top of the file**

When you add `'use client'` to a file, that component and ALL components it imports become Client Components:

```tsx
// components/ProductCard.tsx
'use client';  // This file and everything it imports is client-side

import { useState } from 'react';
import ProductImage from './ProductImage';  // Also becomes Client Component
import ProductPrice from './ProductPrice';  // Also becomes Client Component

export default function ProductCard() {
  const [isHovered, setIsHovered] = useState(false);
  // ...
}
```

**Rule 2: Server Components can import Client Components**

This is the key to the hybrid architecture:

```tsx
// app/products/page.tsx - Server Component
import ProductCard from '@/components/ProductCard';  // Client Component

export default async function ProductsPage() {
  const products = await fetchProducts();  // Server-side fetch
  
  return (
    <div>
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

**Rule 3: Client Components CANNOT import Server Components**

This would fail:

```tsx
// components/ProductCard.tsx - Client Component
'use client';

import ProductDetails from './ProductDetails';  // If this is a Server Component, ERROR

export default function ProductCard() {
  // ...
}
```

**Error**:
```bash
Error: Server Component cannot be imported into Client Component.
```

**Rule 4: But you CAN pass Server Components as props**

This works:

```tsx
// components/Modal.tsx - Client Component
'use client';

import { ReactNode } from 'react';

interface ModalProps {
  children: ReactNode;  // Can be a Server Component!
}

export default function Modal({ children }: ModalProps) {
  return (
    <div className="modal">
      {children}
    </div>
  );
}
```

```tsx
// app/products/[id]/page.tsx - Server Component
import Modal from '@/components/Modal';

async function getProduct(id: string) {
  // Server-side fetch
}

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id);
  
  return (
    <Modal>
      {/* This content is a Server Component, passed as children */}
      <div>
        <h1>{product.name}</h1>
        <p>{product.description}</p>
      </div>
    </Modal>
  );
}
```

### Common Failure Modes and Their Signatures

#### Symptom: "Cannot use hooks in Server Component"

**Browser behavior**: Build fails, app won't start

**Terminal Output**:
```bash
Error: useState can only be used in Client Components.
Add 'use client' to the top of the file.
```

**Root cause**: Trying to use `useState`, `useEffect`, or other hooks in a Server Component

**Solution**: Add `'use client'` directive to the file, or move the stateful logic to a separate Client Component

#### Symptom: "Event handlers cannot be passed to Client Component props"

**Browser behavior**: Build fails

**Terminal Output**:
```bash
Error: Event handlers cannot be passed to Client Component props.
  <button onClick={function} ...>
```

**Root cause**: Trying to pass a function (event handler) from Server Component to Client Component

**Solution**: Move the event handler into the Client Component itself

#### Symptom: "Cannot access process.env in Client Component"

**Browser behavior**: `process.env.API_KEY` is `undefined` in browser

**Browser Console**:
```
Warning: process.env.API_KEY is undefined
```

**Root cause**: Environment variables are not available in Client Components (they run in the browser)

**Solution**: 
- Use Server Components for server-only code
- Or prefix with `NEXT_PUBLIC_` to expose to browser (only for non-secret values)

### When to Apply: Server vs. Client Component Decision Framework

**Use Server Components when**:
- Fetching data from databases or APIs
- Accessing server-only resources (file system, environment variables)
- Keeping sensitive logic on the server (API keys, business logic)
- Reducing JavaScript bundle size
- Improving SEO (content in initial HTML)

**Use Client Components when**:
- Using React hooks (`useState`, `useEffect`, etc.)
- Handling browser events (`onClick`, `onChange`, etc.)
- Using browser-only APIs (`localStorage`, `window`, etc.)
- Using libraries that depend on browser APIs

**Hybrid approach** (most common):
- Server Component for data fetching and layout
- Client Components for interactive pieces (buttons, forms, modals)
- Pass data from Server to Client via props

**Code characteristics**:
- **Server Components**: 
  - Setup complexity: Low
  - Maintenance burden: Low
  - Performance impact: Positive (smaller bundles, faster loads)
  - SEO impact: Positive (content in HTML)

- **Client Components**:
  - Setup complexity: Low (just add `'use client'`)
  - Maintenance burden: Medium (manage state, effects)
  - Performance impact: Neutral to negative (larger bundles)
  - Interactivity: Required for user interactions

**Limitation preview**: We now understand Server and Client Components, but we haven't addressed how to share UI across multiple pages (navigation, footers) or handle loading states during data fetching. That's what layouts and special files solve, which we'll explore next.

## Layouts and templates

## Layouts and templates

Every application needs shared UI—navigation bars, footers, sidebars. In client-only React, we manually wrapped pages with these components. Next.js provides a better way: layouts that automatically wrap all pages in a route segment.

Layouts are Server Components by default, which means they can fetch data once and share it across multiple pages. They also preserve state during navigation, avoiding unnecessary re-renders.

### The Problem: Repeating Navigation on Every Page

Let's see what happens when we manually add navigation to each page:

```tsx
// app/page.tsx - Home page with navigation
import Link from 'next/link';

export default function HomePage() {
  return (
    <>
      <nav className="bg-gray-800 text-white p-4">
        <div className="container mx-auto flex gap-4">
          <Link href="/" className="hover:text-gray-300">Home</Link>
          <Link href="/products" className="hover:text-gray-300">Products</Link>
          <Link href="/cart" className="hover:text-gray-300">Cart</Link>
        </div>
      </nav>
      
      <div className="container mx-auto px-4 py-8">
        <h1 className="text-4xl font-bold mb-4">Welcome to TechStore</h1>
        <p className="text-lg text-gray-600">
          Discover the latest in technology and gadgets
        </p>
      </div>
    </>
  );
}
```

```tsx
// app/products/page.tsx - Products page with navigation
import Link from 'next/link';

export default function ProductsPage() {
  const products = [
    { id: '1', name: 'Wireless Headphones', price: 99.99 },
    { id: '2', name: 'Smart Watch', price: 299.99 },
    { id: '3', name: 'Laptop Stand', price: 49.99 },
  ];

  return (
    <>
      {/* Exact same navigation code repeated */}
      <nav className="bg-gray-800 text-white p-4">
        <div className="container mx-auto flex gap-4">
          <Link href="/" className="hover:text-gray-300">Home</Link>
          <Link href="/products" className="hover:text-gray-300">Products</Link>
          <Link href="/cart" className="hover:text-gray-300">Cart</Link>
        </div>
      </nav>
      
      <div className="container mx-auto px-4 py-8">
        <h1 className="text-3xl font-bold mb-6">Products</h1>
        <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
          {products.map((product) => (
            <div key={product.id} className="border rounded-lg p-4">
              <h2 className="text-xl font-semibold mb-2">{product.name}</h2>
              <p className="text-gray-600 mb-4">${product.price}</p>
              <Link 
                href={`/products/${product.id}`}
                className="text-blue-600 hover:underline"
              >
                View Details
              </Link>
            </div>
          ))}
        </div>
      </div>
    </>
  );
}
```

**Problems with this approach**:

1. **Code duplication**: Navigation code repeated in every page
2. **Maintenance burden**: Update navigation in one place, must update everywhere
3. **Re-render on navigation**: Navigation re-renders even though it hasn't changed
4. **No state preservation**: If navigation had state (e.g., mobile menu open), it would reset on every page change

**Diagnostic Analysis: Reading the Re-render Evidence**

**React DevTools - Profiler**:
1. Start recording
2. Navigate from Home to Products
3. Stop recording
4. Observe: Navigation component rendered twice (once for each page)

**Browser behavior**:
- Navigate between pages
- Notice: Navigation briefly flickers (re-renders)
- If navigation had animations or state, they would reset

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Navigation stays stable during page transitions
   - Actual: Subtle flicker, any navigation state would be lost

2. **What React DevTools reveals**:
   - Key indicator: Navigation component in both page trees
   - This means: Navigation is part of each page, not shared
   - Evidence: Two separate instances, not one preserved instance

3. **Root cause identified**: Navigation is duplicated in each page component

4. **Why the current approach can't solve this**: Manual duplication can't preserve component instances across routes

5. **What we need**: A way to define shared UI once that wraps multiple pages

### Iteration 4: Root Layout

Next.js provides `layout.tsx` files that automatically wrap all pages in their route segment:

```tsx
// app/layout.tsx - Root layout (wraps ALL pages)
import Link from 'next/link';
import './globals.css';

export const metadata = {
  title: 'TechStore',
  description: 'Your one-stop shop for technology',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        {/* Navigation is now defined once */}
        <nav className="bg-gray-800 text-white p-4">
          <div className="container mx-auto flex gap-4">
            <Link href="/" className="hover:text-gray-300">Home</Link>
            <Link href="/products" className="hover:text-gray-300">Products</Link>
            <Link href="/cart" className="hover:text-gray-300">Cart</Link>
          </div>
        </nav>
        
        {/* Page content renders here */}
        <main>{children}</main>
        
        {/* Footer also defined once */}
        <footer className="bg-gray-800 text-white p-4 mt-8">
          <div className="container mx-auto text-center">
            <p>© 2025 TechStore. All rights reserved.</p>
          </div>
        </footer>
      </body>
    </html>
  );
}
```

Now simplify the pages:

```tsx
// app/page.tsx - Home page (no navigation needed)
export default function HomePage() {
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-4xl font-bold mb-4">Welcome to TechStore</h1>
      <p className="text-lg text-gray-600">
        Discover the latest in technology and gadgets
      </p>
    </div>
  );
}
```

```tsx
// app/products/page.tsx - Products page (no navigation needed)
import Link from 'next/link';

export default function ProductsPage() {
  const products = [
    { id: '1', name: 'Wireless Headphones', price: 99.99 },
    { id: '2', name: 'Smart Watch', price: 299.99 },
    { id: '3', name: 'Laptop Stand', price: 49.99 },
  ];

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-6">Products</h1>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {products.map((product) => (
          <div key={product.id} className="border rounded-lg p-4">
            <h2 className="text-xl font-semibold mb-2">{product.name}</h2>
            <p className="text-gray-600 mb-4">${product.price}</p>
            <Link 
              href={`/products/${product.id}`}
              className="text-blue-600 hover:underline"
            >
              View Details
            </Link>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Verification**: Navigate between pages and watch React DevTools.

**React DevTools - Profiler** (After fix):
1. Start recording
2. Navigate from Home to Products
3. Stop recording
4. Observe: Navigation component NOT in the render (it's preserved)
5. Only the page content (`children`) re-renders

**Browser behavior**:
- Navigate between pages
- Notice: Navigation stays completely stable, no flicker
- Footer also stays stable

**Expected vs. Actual improvement**: Navigation and footer no longer re-render during page transitions. Code is defined once, maintained in one place.

### Nested Layouts: Different Sections, Different UI

Sometimes different sections of your app need different layouts. For example, a dashboard might have a sidebar, while marketing pages don't.

**Project Structure**:

```bash
app/
├── layout.tsx                 # Root layout (navigation + footer)
├── page.tsx                   # Home page
├── products/
│   ├── layout.tsx            # Products layout (adds breadcrumbs)
│   ├── page.tsx              # Products list
│   └── [id]/
│       └── page.tsx          # Product detail
└── dashboard/
    ├── layout.tsx            # Dashboard layout (adds sidebar)
    ├── page.tsx              # Dashboard home
    └── settings/
        └── page.tsx          # Dashboard settings
```

Let's add a products layout with breadcrumbs:

```tsx
// app/products/layout.tsx - Products section layout
import Link from 'next/link';

export default function ProductsLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      {/* Breadcrumbs for products section */}
      <div className="bg-gray-100 py-2">
        <div className="container mx-auto px-4">
          <nav className="text-sm text-gray-600">
            <Link href="/" className="hover:text-gray-900">Home</Link>
            <span className="mx-2">/</span>
            <Link href="/products" className="hover:text-gray-900">Products</Link>
          </nav>
        </div>
      </div>
      
      {/* Page content */}
      {children}
    </div>
  );
}
```

Now every page under `/products` automatically gets breadcrumbs:
- `/products` → Home / Products
- `/products/1` → Home / Products (breadcrumbs from layout)

**Layout nesting**:
1. Root layout (`app/layout.tsx`) wraps everything
2. Products layout (`app/products/layout.tsx`) wraps products pages
3. Page content renders inside both layouts

**Visual hierarchy**:
```
<RootLayout>           ← Navigation + Footer
  <ProductsLayout>     ← Breadcrumbs
    <ProductsPage />   ← Actual page content
  </ProductsLayout>
</RootLayout>
```

### The Failure: Layout Re-renders When It Shouldn't

Let's add a search input to the products layout:

```tsx
// app/products/layout.tsx - With search (PROBLEMATIC)
'use client';  // Need client component for state

import { useState } from 'react';
import Link from 'next/link';

export default function ProductsLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const [search, setSearch] = useState('');

  return (
    <div>
      <div className="bg-gray-100 py-2">
        <div className="container mx-auto px-4 flex justify-between items-center">
          <nav className="text-sm text-gray-600">
            <Link href="/" className="hover:text-gray-900">Home</Link>
            <span className="mx-2">/</span>
            <Link href="/products" className="hover:text-gray-900">Products</Link>
          </nav>
          
          <input
            type="text"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            placeholder="Search products..."
            className="px-3 py-1 border rounded"
          />
        </div>
      </div>
      
      {children}
    </div>
  );
}
```

**Diagnostic Analysis: Reading the Layout Re-render**

**Browser behavior**:
1. Navigate to `/products`
2. Type "headphones" in search box
3. Click on a product to go to `/products/1`
4. Observe: Search box is empty (state was lost)

**React DevTools - Components Tab**:
- Navigate between products pages
- Observe: `ProductsLayout` component unmounts and remounts
- State resets on every navigation

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Search input preserves text during navigation
   - Actual: Search input clears when navigating to product detail

2. **What React DevTools reveals**:
   - Key indicator: Layout component unmounts/remounts
   - This means: Layout is treated as part of the page, not preserved
   - Evidence: State resets, component tree rebuilds

3. **Root cause identified**: Making the layout a Client Component causes it to re-render with each page change

4. **Why the current approach can't solve this**: Client Component layouts don't preserve state across navigations in the same way Server Component layouts do

5. **What we need**: Keep layout as Server Component, move interactive parts to separate Client Components

### Iteration 5: Hybrid Layout with Client Component

Extract the search input into its own Client Component:

```tsx
// app/products/SearchBar.tsx - Client Component
'use client';

import { useState } from 'react';

export default function SearchBar() {
  const [search, setSearch] = useState('');

  return (
    <input
      type="text"
      value={search}
      onChange={(e) => setSearch(e.target.value)}
      placeholder="Search products..."
      className="px-3 py-1 border rounded"
    />
  );
}
```

```tsx
// app/products/layout.tsx - Server Component layout
import Link from 'next/link';
import SearchBar from './SearchBar';

export default function ProductsLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      <div className="bg-gray-100 py-2">
        <div className="container mx-auto px-4 flex justify-between items-center">
          <nav className="text-sm text-gray-600">
            <Link href="/" className="hover:text-gray-900">Home</Link>
            <span className="mx-2">/</span>
            <Link href="/products" className="hover:text-gray-900">Products</Link>
          </nav>
          
          {/* Client Component for interactivity */}
          <SearchBar />
        </div>
      </div>
      
      {children}
    </div>
  );
}
```

**Verification**: Test the search input across navigation.

**Browser behavior**:
1. Navigate to `/products`
2. Type "headphones" in search box
3. Click on a product to go to `/products/1`
4. Observe: Search box STILL contains "headphones" (state preserved)

**React DevTools - Components Tab**:
- `ProductsLayout` is not visible (Server Component)
- `SearchBar` is visible and maintains state across navigations

**Expected vs. Actual improvement**: Search input now preserves state during navigation because it's a separate Client Component that Next.js keeps mounted.

### Templates: When You DO Want to Re-render

Sometimes you want layout-like behavior but WITH re-rendering on navigation. Use `template.tsx`:

```tsx
// app/products/template.tsx - Re-renders on every navigation
'use client';

import { useEffect } from 'react';

export default function ProductsTemplate({
  children,
}: {
  children: React.ReactNode;
}) {
  useEffect(() => {
    // This runs on EVERY navigation
    console.log('Products template mounted');
    
    // Example: Reset scroll position
    window.scrollTo(0, 0);
    
    // Example: Track page view
    // analytics.track('page_view');
  }, []);

  return <div>{children}</div>;
}
```

**Difference between layout and template**:

| Feature | `layout.tsx` | `template.tsx` |
|---------|-------------|----------------|
| Re-renders on navigation | No | Yes |
| Preserves state | Yes | No |
| Use for | Navigation, shared UI | Analytics, scroll reset |
| Performance | Better (no re-render) | Worse (re-renders) |

**When to use templates**:
- Resetting scroll position on navigation
- Triggering analytics on every page view
- Animations that should replay on navigation
- Any side effect that should run on every route change

### Route Groups with Different Layouts

Remember route groups from section 16.1? They're especially useful with layouts:

```bash
app/
├── layout.tsx                 # Root layout (all pages)
├── (marketing)/
│   ├── layout.tsx            # Marketing layout (no sidebar)
│   ├── page.tsx              # Home (/)
│   ├── about/
│   │   └── page.tsx          # About (/about)
│   └── contact/
│       └── page.tsx          # Contact (/contact)
└── (shop)/
    ├── layout.tsx            # Shop layout (with cart widget)
    ├── products/
    │   └── page.tsx          # Products (/products)
    └── cart/
        └── page.tsx          # Cart (/cart)
```

```tsx
// app/(marketing)/layout.tsx - Marketing layout
export default function MarketingLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="max-w-4xl mx-auto">
      {/* Centered content, no sidebar */}
      {children}
    </div>
  );
}
```

```tsx
// app/(shop)/layout.tsx - Shop layout
import CartWidget from '@/components/CartWidget';

export default function ShopLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex">
      {/* Main content */}
      <div className="flex-1">{children}</div>
      
      {/* Sticky cart widget */}
      <aside className="w-64 sticky top-0 h-screen">
        <CartWidget />
      </aside>
    </div>
  );
}
```

Now:
- Marketing pages (`/`, `/about`, `/contact`) use centered layout
- Shop pages (`/products`, `/cart`) use layout with cart widget
- Both share the root layout (navigation + footer)

### Metadata in Layouts

Layouts can also define metadata (title, description, Open Graph tags):

```tsx
// app/layout.tsx - Root layout with metadata
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: {
    default: 'TechStore',
    template: '%s | TechStore',  // Page title | TechStore
  },
  description: 'Your one-stop shop for technology',
  openGraph: {
    title: 'TechStore',
    description: 'Your one-stop shop for technology',
    images: ['/og-image.jpg'],
  },
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

```tsx
// app/products/[id]/page.tsx - Page with specific metadata
import type { Metadata } from 'next';

interface ProductPageProps {
  params: { id: string };
}

export async function generateMetadata({ 
  params 
}: ProductPageProps): Promise<Metadata> {
  // Fetch product to get title
  const product = await getProduct(params.id);
  
  return {
    title: product.name,  // Uses template: "Wireless Headphones | TechStore"
    description: product.description,
  };
}

export default async function ProductPage({ params }: ProductPageProps) {
  const product = await getProduct(params.id);
  // ...
}
```

### When to Apply: Layout vs. Template Decision Framework

**Use `layout.tsx` when**:
- Defining shared UI (navigation, sidebars, footers)
- You want to preserve state during navigation
- You want to avoid unnecessary re-renders
- Most common use case

**Use `template.tsx` when**:
- You need side effects on every navigation (analytics, scroll reset)
- You want animations to replay on navigation
- You explicitly want state to reset
- Less common, specific use cases

**Use route groups when**:
- Different sections need different layouts
- You want to organize code without affecting URLs
- You're building a large app with distinct sections

**Code characteristics**:
- **Layouts**:
  - Setup complexity: Low (just create `layout.tsx`)
  - Maintenance burden: Low (define once, applies to all child routes)
  - Performance impact: Positive (no re-render on navigation)
  - State preservation: Yes

- **Templates**:
  - Setup complexity: Low (just create `template.tsx`)
  - Maintenance burden: Low
  - Performance impact: Negative (re-renders on every navigation)
  - State preservation: No

**Limitation preview**: We now have shared layouts, but we haven't addressed what happens during data fetching (loading states) or when errors occur. That's what special files like `loading.tsx` and `error.tsx` solve, which we'll explore next.

## Loading and error states

## Loading and error states

In client-only React, we manually managed loading and error states with `useState` and `useEffect`. Next.js provides a better way: special files that automatically handle these states at the route level.

This isn't just about convenience—it enables **streaming**, where parts of your page can load independently, showing content as soon as it's ready rather than waiting for everything.

### The Problem: Manual Loading States Don't Scale

Let's see what happens when we fetch data without proper loading states:

```tsx
// app/products/[id]/page.tsx - No loading state (PROBLEMATIC)
interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  reviews: Review[];
}

interface Review {
  id: string;
  author: string;
  rating: number;
  comment: string;
}

async function getProduct(id: string): Promise<Product> {
  // Simulate slow API call
  await new Promise(resolve => setTimeout(resolve, 2000));
  
  const res = await fetch(`https://api.example.com/products/${id}`);
  return res.json();
}

export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  const product = await getProduct(params.id);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
      <p className="text-gray-600 mb-4">{product.description}</p>
      <p className="text-2xl font-bold mb-8">${product.price}</p>
      
      <h2 className="text-2xl font-bold mb-4">Reviews</h2>
      <div className="space-y-4">
        {product.reviews.map(review => (
          <div key={review.id} className="border rounded p-4">
            <div className="flex justify-between mb-2">
              <span className="font-semibold">{review.author}</span>
              <span className="text-yellow-500">{'⭐'.repeat(review.rating)}</span>
            </div>
            <p className="text-gray-600">{review.comment}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Diagnostic Analysis: Reading the Loading Void**

**Browser behavior**:
1. Click on a product link
2. Observe: Page goes completely blank
3. Wait 2 seconds (simulated API delay)
4. Observe: Content suddenly appears

**Browser DevTools - Network Tab**:
- Request to `/products/1` starts
- Response takes 2000ms
- During this time: No HTML sent, no content visible
- After 2000ms: Full HTML arrives

**User experience**:
- Expected: Some indication that content is loading
- Actual: Blank white screen, no feedback
- User thinks: "Did my click work? Is the site broken?"

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Loading spinner or skeleton screen
   - Actual: Blank page for 2 seconds, then content appears

2. **What the Network tab reveals**:
   - Key indicator: Single large response after 2 seconds
   - This means: Server waits for ALL data before sending ANY HTML
   - Evidence: No progressive rendering, all-or-nothing approach

3. **Root cause identified**: Server Component waits for `await getProduct()` to complete before rendering anything

4. **Why the current approach can't solve this**: No mechanism to show loading UI while data fetches

5. **What we need**: A way to show loading UI immediately while data fetches in the background

### Iteration 6: Loading UI with `loading.tsx`

Next.js provides `loading.tsx` files that automatically wrap pages in a Suspense boundary:

```tsx
// app/products/[id]/loading.tsx - Loading UI
export default function ProductLoading() {
  return (
    <div className="container mx-auto px-4 py-8">
      {/* Skeleton for product details */}
      <div className="animate-pulse">
        <div className="h-8 bg-gray-200 rounded w-3/4 mb-4"></div>
        <div className="h-4 bg-gray-200 rounded w-full mb-2"></div>
        <div className="h-4 bg-gray-200 rounded w-5/6 mb-4"></div>
        <div className="h-6 bg-gray-200 rounded w-24 mb-8"></div>
        
        {/* Skeleton for reviews */}
        <div className="h-6 bg-gray-200 rounded w-32 mb-4"></div>
        <div className="space-y-4">
          {[1, 2, 3].map(i => (
            <div key={i} className="border rounded p-4">
              <div className="h-4 bg-gray-200 rounded w-1/4 mb-2"></div>
              <div className="h-4 bg-gray-200 rounded w-full"></div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

The page component stays the same—Next.js automatically shows `loading.tsx` while the page loads:

```tsx
// app/products/[id]/page.tsx - Same as before
async function getProduct(id: string): Promise<Product> {
  await new Promise(resolve => setTimeout(resolve, 2000));
  const res = await fetch(`https://api.example.com/products/${id}`);
  return res.json();
}

export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  const product = await getProduct(params.id);
  // ... same rendering code
}
```

**Verification**: Click on a product and watch the loading state.

**Browser behavior**:
1. Click on a product link
2. Observe: Skeleton screen appears IMMEDIATELY
3. Wait 2 seconds
4. Observe: Skeleton smoothly transitions to actual content

**Browser DevTools - Network Tab**:
- Initial HTML arrives immediately (~50ms)
- Contains the loading skeleton
- After 2000ms: Streaming update with actual product data

**User experience**:
- Expected: Immediate feedback that something is happening
- Actual: Skeleton screen shows instantly, content loads progressively

**Expected vs. Actual improvement**: User sees feedback immediately instead of staring at a blank screen. Perceived performance is much better even though actual load time is the same.

### The Failure: Slow Component Blocks Everything

Our product page has two data sources: product details and reviews. Currently, if reviews are slow, the entire page waits:

```tsx
// app/products/[id]/page.tsx - Everything waits for slowest data
async function getProduct(id: string): Promise<Product> {
  await new Promise(resolve => setTimeout(resolve, 500));  // Fast
  const res = await fetch(`https://api.example.com/products/${id}`);
  return res.json();
}

async function getReviews(productId: string): Promise<Review[]> {
  await new Promise(resolve => setTimeout(resolve, 3000));  // SLOW
  const res = await fetch(`https://api.example.com/reviews?productId=${productId}`);
  return res.json();
}

export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  // Both fetches happen sequentially
  const product = await getProduct(params.id);
  const reviews = await getReviews(params.id);

  return (
    <div className="container mx-auto px-4 py-8">
      {/* Product details - ready after 500ms */}
      <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
      <p className="text-gray-600 mb-4">{product.description}</p>
      <p className="text-2xl font-bold mb-8">${product.price}</p>
      
      {/* Reviews - not ready until 3500ms (500 + 3000) */}
      <h2 className="text-2xl font-bold mb-4">Reviews</h2>
      <div className="space-y-4">
        {reviews.map(review => (
          <div key={review.id} className="border rounded p-4">
            <div className="flex justify-between mb-2">
              <span className="font-semibold">{review.author}</span>
              <span className="text-yellow-500">{'⭐'.repeat(review.rating)}</span>
            </div>
            <p className="text-gray-600">{review.comment}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Diagnostic Analysis: Reading the Sequential Waterfall**

**Browser DevTools - Network Tab**:
1. Request to `/products/1` starts
2. Server fetches product (500ms)
3. Server fetches reviews (3000ms)
4. Total time: 3500ms before ANY content appears

**Timeline**:
```
0ms     ─────────────────────────────────────────────────────────────
        Loading skeleton visible
        
500ms   Product data ready (but not shown yet)
        
3500ms  Reviews data ready
        ─────────────────────────────────────────────────────────────
        Full page appears
```

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Product details appear quickly, reviews load separately
   - Actual: Everything waits for slowest component (reviews)

2. **What the Network tab reveals**:
   - Key indicator: Single response after 3500ms
   - This means: Server waits for ALL data before sending ANY content
   - Evidence: Product details (ready at 500ms) are held hostage by reviews (ready at 3500ms)

3. **Root cause identified**: Sequential `await` statements block each other

4. **Why the current approach can't solve this**: Can't show partial content while waiting for slow data

5. **What we need**: A way to stream content as it becomes ready

### Iteration 7: Streaming with Suspense

React's `Suspense` component allows parts of the page to load independently:

```tsx
// app/products/[id]/page.tsx - Streaming with Suspense
import { Suspense } from 'react';

async function getProduct(id: string): Promise<Product> {
  await new Promise(resolve => setTimeout(resolve, 500));
  const res = await fetch(`https://api.example.com/products/${id}`);
  return res.json();
}

async function getReviews(productId: string): Promise<Review[]> {
  await new Promise(resolve => setTimeout(resolve, 3000));
  const res = await fetch(`https://api.example.com/reviews?productId=${productId}`);
  return res.json();
}

// Extract reviews into separate component
async function ProductReviews({ productId }: { productId: string }) {
  const reviews = await getReviews(productId);
  
  return (
    <div className="space-y-4">
      {reviews.map(review => (
        <div key={review.id} className="border rounded p-4">
          <div className="flex justify-between mb-2">
            <span className="font-semibold">{review.author}</span>
            <span className="text-yellow-500">{'⭐'.repeat(review.rating)}</span>
          </div>
          <p className="text-gray-600">{review.comment}</p>
        </div>
      ))}
    </div>
  );
}

// Loading fallback for reviews
function ReviewsSkeleton() {
  return (
    <div className="space-y-4 animate-pulse">
      {[1, 2, 3].map(i => (
        <div key={i} className="border rounded p-4">
          <div className="h-4 bg-gray-200 rounded w-1/4 mb-2"></div>
          <div className="h-4 bg-gray-200 rounded w-full"></div>
        </div>
      ))}
    </div>
  );
}

export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  // Only wait for product details
  const product = await getProduct(params.id);

  return (
    <div className="container mx-auto px-4 py-8">
      {/* Product details - shows after 500ms */}
      <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
      <p className="text-gray-600 mb-4">{product.description}</p>
      <p className="text-2xl font-bold mb-8">${product.price}</p>
      
      {/* Reviews - streams in after 3000ms */}
      <h2 className="text-2xl font-bold mb-4">Reviews</h2>
      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews productId={params.id} />
      </Suspense>
    </div>
  );
}
```

**Verification**: Click on a product and watch the streaming behavior.

**Browser behavior**:
1. Click on a product link
2. Observe: Page skeleton appears immediately
3. After 500ms: Product details appear, reviews show skeleton
4. After 3000ms more: Reviews appear

**Browser DevTools - Network Tab**:
- Initial HTML arrives at ~500ms (with product details)
- Streaming update arrives at ~3500ms (with reviews)
- Two separate chunks, not one monolithic response

**Timeline** (After fix):
```
0ms     ─────────────────────────────────────────────────────────────
        Loading skeleton visible
        
500ms   Product details appear ✓
        Reviews skeleton visible
        
3500ms  Reviews appear ✓
        ─────────────────────────────────────────────────────────────
```

**Performance Metrics**:
- **Before** (no Suspense):
  - Time to first content: 3500ms
  - User sees: Nothing until everything is ready
  
- **After** (with Suspense):
  - Time to first content: 500ms (7x faster)
  - Time to full content: 3500ms (same)
  - User sees: Product details immediately, reviews load progressively

**Expected vs. Actual improvement**: User can start reading product details 3 seconds earlier. Perceived performance is dramatically better.

### Error Handling with `error.tsx`

Just like `loading.tsx`, Next.js provides `error.tsx` for automatic error boundaries:

```tsx
// app/products/[id]/error.tsx - Error UI
'use client';  // Error components must be Client Components

import { useEffect } from 'react';

export default function ProductError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Log error to error reporting service
    console.error('Product page error:', error);
  }, [error]);

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="bg-red-50 border border-red-200 rounded-lg p-6">
        <h2 className="text-2xl font-bold text-red-800 mb-4">
          Something went wrong!
        </h2>
        <p className="text-red-600 mb-4">
          {error.message || 'Failed to load product'}
        </p>
        <button
          onClick={reset}
          className="bg-red-600 text-white px-4 py-2 rounded hover:bg-red-700"
        >
          Try again
        </button>
      </div>
    </div>
  );
}
```

Now let's simulate an error:

```tsx
// app/products/[id]/page.tsx - With error handling
async function getProduct(id: string): Promise<Product> {
  await new Promise(resolve => setTimeout(resolve, 500));
  
  const res = await fetch(`https://api.example.com/products/${id}`);
  
  if (!res.ok) {
    throw new Error('Product not found');
  }
  
  return res.json();
}

export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  // If this throws, error.tsx catches it
  const product = await getProduct(params.id);
  
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
      <p className="text-gray-600 mb-4">{product.description}</p>
      <p className="text-2xl font-bold mb-8">${product.price}</p>
    </div>
  );
}
```

**Verification**: Navigate to a non-existent product (e.g., `/products/999`).

**Browser behavior**:
1. Navigate to `/products/999`
2. Observe: Loading skeleton appears
3. After 500ms: Error UI appears with "Product not found" message
4. Click "Try again" → Retries the request

**Browser Console**:
```
Product page error: Error: Product not found
    at getProduct (page.tsx:8)
```

**React DevTools - Components Tab**:
- `ProductError` component visible
- Props: `{ error: Error, reset: function }`

### Error Boundaries at Different Levels

You can have error boundaries at multiple levels:

```bash
app/
├── error.tsx                  # Catches errors in entire app
├── products/
│   ├── error.tsx             # Catches errors in products section
│   └── [id]/
│       ├── error.tsx         # Catches errors in specific product
│       └── page.tsx
```

More specific error boundaries take precedence. This allows you to:
- Show generic error UI at the root level
- Show section-specific error UI (e.g., "Products unavailable")
- Show page-specific error UI (e.g., "Product not found")

### The Failure: Error in Suspense Boundary

What happens when a component inside `Suspense` throws an error?

```tsx
// app/products/[id]/page.tsx - Error in Suspense boundary
async function ProductReviews({ productId }: { productId: string }) {
  const reviews = await getReviews(productId);
  
  if (reviews.length === 0) {
    throw new Error('No reviews available');
  }
  
  return (
    <div className="space-y-4">
      {reviews.map(review => (
        <div key={review.id} className="border rounded p-4">
          <div className="flex justify-between mb-2">
            <span className="font-semibold">{review.author}</span>
            <span className="text-yellow-500">{'⭐'.repeat(review.rating)}</span>
          </div>
          <p className="text-gray-600">{review.comment}</p>
        </div>
      ))}
    </div>
  );
}

export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  const product = await getProduct(params.id);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
      <p className="text-gray-600 mb-4">{product.description}</p>
      <p className="text-2xl font-bold mb-8">${product.price}</p>
      
      <h2 className="text-2xl font-bold mb-4">Reviews</h2>
      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews productId={params.id} />
      </Suspense>
    </div>
  );
}
```

**Browser behavior**:
1. Navigate to product with no reviews
2. Observe: Product details appear
3. Observe: Reviews section shows error UI from `error.tsx`
4. Product details remain visible (not affected by reviews error)

**Key insight**: Errors inside `Suspense` boundaries are caught by the nearest error boundary, but don't affect sibling content. This is **error isolation**—one failing component doesn't crash the entire page.

### Iteration 8: Granular Error Boundaries

For better error isolation, wrap Suspense boundaries in their own error handlers:

```tsx
// app/products/[id]/ReviewsErrorBoundary.tsx
'use client';

export default function ReviewsErrorBoundary({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div className="border border-red-200 rounded-lg p-4 bg-red-50">
      <p className="text-red-600 mb-2">Failed to load reviews</p>
      <button
        onClick={reset}
        className="text-sm text-red-600 hover:text-red-800 underline"
      >
        Try again
      </button>
    </div>
  );
}
```

```tsx
// app/products/[id]/page.tsx - With granular error boundary
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';
import ReviewsErrorBoundary from './ReviewsErrorBoundary';

export default async function ProductPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  const product = await getProduct(params.id);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
      <p className="text-gray-600 mb-4">{product.description}</p>
      <p className="text-2xl font-bold mb-8">${product.price}</p>
      
      <h2 className="text-2xl font-bold mb-4">Reviews</h2>
      <ErrorBoundary FallbackComponent={ReviewsErrorBoundary}>
        <Suspense fallback={<ReviewsSkeleton />}>
          <ProductReviews productId={params.id} />
        </Suspense>
      </ErrorBoundary>
    </div>
  );
}
```

Now reviews errors show a small inline error message instead of replacing the entire page.

### Common Failure Modes and Their Signatures

#### Symptom: Entire page shows loading state for too long

**Browser behavior**: Loading skeleton visible for 5+ seconds

**Network tab**: Single large response after long delay

**Root cause**: Not using Suspense to stream slow components

**Solution**: Wrap slow components in `Suspense` boundaries

#### Symptom: Error in one component crashes entire page

**Browser behavior**: Whole page replaced with error UI

**React DevTools**: Error boundary at root level caught the error

**Root cause**: No granular error boundaries

**Solution**: Add `error.tsx` files at appropriate route segments, or use `ErrorBoundary` components

#### Symptom: Loading state never disappears

**Browser behavior**: Skeleton screen stays forever

**Browser Console**:
```
Warning: A component suspended while responding to synchronous input.
```

**Root cause**: Component inside Suspense is stuck (infinite loop, never resolves)

**Solution**: Check async function for proper error handling and resolution

### When to Apply: Loading and Error State Decision Framework

**Use `loading.tsx` when**:
- You want automatic loading UI for an entire route segment
- You're okay with the same loading UI for all pages in that segment
- You want the simplest possible setup

**Use `Suspense` when**:
- You want to stream different parts of the page independently
- You have slow and fast data sources on the same page
- You want granular control over loading states
- You want to show partial content while other parts load

**Use `error.tsx` when**:
- You want automatic error handling for an entire route segment
- You want consistent error UI across multiple pages
- You want the simplest possible setup

**Use `ErrorBoundary` when**:
- You want granular error handling for specific components
- You want different error UIs for different failures
- You want to isolate errors (one component fails, others stay)

**Code characteristics**:
- **`loading.tsx`**:
  - Setup complexity: Minimal (just create file)
  - Granularity: Route segment level
  - Performance impact: Positive (shows feedback immediately)

- **`Suspense`**:
  - Setup complexity: Low (wrap components)
  - Granularity: Component level
  - Performance impact: Very positive (streaming, progressive rendering)

- **`error.tsx`**:
  - Setup complexity: Minimal (just create file)
  - Granularity: Route segment level
  - Error isolation: Moderate

- **`ErrorBoundary`**:
  - Setup complexity: Low (wrap components)
  - Granularity: Component level
  - Error isolation: Excellent

### The Complete Journey: From Blank Screen to Streaming

| Iteration | Problem | Solution | Result |
|-----------|---------|----------|--------|
| 0 | Blank screen during load | None | Poor UX, no feedback |
| 6 | Need loading feedback | `loading.tsx` | Skeleton screen |
| 7 | Slow component blocks page | `Suspense` | Progressive rendering |
| 8 | Error crashes entire page | Granular error boundaries | Error isolation |

**Final Implementation**: Our product page now:
- Shows loading skeleton immediately
- Streams product details as soon as ready (500ms)
- Streams reviews independently (3500ms)
- Handles errors gracefully without crashing the page
- Provides excellent perceived performance

**Decision Framework**: When building a page:
1. Start with `loading.tsx` for basic loading UI
2. Add `Suspense` for components with slow data
3. Add `error.tsx` for route-level error handling
4. Add granular `ErrorBoundary` for critical components that should fail independently

**Lessons Learned**:
- Loading states are not optional—they're essential for good UX
- Streaming with Suspense dramatically improves perceived performance
- Error boundaries prevent one failure from crashing the entire app
- Granular boundaries (both Suspense and error) provide the best user experience
