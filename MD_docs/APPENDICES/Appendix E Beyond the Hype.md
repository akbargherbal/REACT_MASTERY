# Chapter E: Beyond the Hype

## React's place in the broader ecosystem

## React's place in the broader ecosystem

You've spent hundreds of pages learning React and Next.js. You've built components, managed state, optimized performance, and deployed to production. Now it's time for the uncomfortable truth: **React is not the answer to every problem**.

This isn't a betrayal of everything you've learned. It's the mark of a mature developer: knowing when your favorite tool is the right choice, and when it's not.

### The Web Development Landscape in 2025

Let's establish where React actually sits in the modern web ecosystem. This isn't about declaring winners and losers—it's about understanding the trade-offs each approach optimizes for.

**The Current Ecosystem**:

```
┌─────────────────────────────────────────────────────────────┐
│                    Web Development Spectrum                  │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Static HTML/CSS ──→ Progressive Enhancement ──→ SPAs        │
│  (Hugo, Jekyll)      (Astro, Eleventy)          (React, Vue) │
│                                                               │
│  ↓                   ↓                           ↓            │
│  Zero JS             Minimal JS                  Heavy JS    │
│  Perfect SEO         Great SEO                   Needs SSR   │
│  Instant load        Fast load                   Slow initial│
│  No interactivity    Selective interactivity     Full app    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

React lives on the right side of this spectrum. It's optimized for **building complex, interactive applications**. But that optimization comes with costs.

### What React Actually Solves

Let's be precise about React's value proposition. React excels when you need:

**1. Complex, Stateful Interactions**

React's component model and state management shine when building interfaces where:
- User actions trigger cascading updates across multiple UI elements
- State needs to be synchronized between distant parts of the component tree
- The UI needs to respond to real-time data changes

**Example scenarios where React excels**:
- Collaborative editing tools (Google Docs, Figma)
- Real-time dashboards with live data updates
- Complex forms with interdependent fields and validation
- E-commerce sites with shopping carts, filters, and dynamic pricing
- Social media feeds with infinite scroll and optimistic updates

**2. Component Reusability at Scale**

React's component model becomes valuable when:
- You're building a design system used across multiple products
- You need to maintain consistency across a large application
- Different teams need to share UI components
- You're building a component library for external consumption

**3. Rich Developer Ecosystem**

React's maturity provides:
- Battle-tested solutions for common problems (React Query, Zustand, React Hook Form)
- Extensive TypeScript support
- Robust debugging tools (React DevTools)
- Large community for troubleshooting
- Abundant learning resources and hiring pool

### What React Doesn't Solve (And the Costs It Imposes)

**The JavaScript Tax**

Every React application starts with a baseline cost:

```
Minimum React Bundle (production, gzipped):
- React core: ~6 KB
- React DOM: ~40 KB
- Your application code: 50-500+ KB
- Total: 96+ KB before your app does anything

Time to Interactive (TTI):
- Download: 100-500ms (on 3G)
- Parse/compile: 200-800ms
- Hydration: 100-500ms
- Total: 400-1800ms before interactive
```

For a blog post or marketing page, this is **pure overhead**. The user waits 1-2 seconds for JavaScript to load and execute before they can click a button—when the same page could have been interactive immediately with plain HTML.

**The Complexity Tax**

React introduces architectural complexity:
- Build tooling (Vite, webpack, Next.js)
- State management decisions (local vs. global vs. server)
- Server/client boundary management (in Next.js)
- Hydration mismatches and debugging
- Performance optimization requirements

For simple sites, this complexity is **accidental**, not essential. You're solving problems React created, not problems your users have.

**The Maintenance Tax**

React's ecosystem moves fast:
- Major version updates every 1-2 years
- Breaking changes in dependencies
- Deprecated patterns (class components, legacy Context API)
- Security vulnerabilities in the dependency tree
- Framework churn (Create React App → Vite → Next.js → ?)

A static HTML site from 2015 still works perfectly. A React app from 2015 requires significant refactoring to run on modern tooling.

### The Honest Assessment: When React Makes Sense

React is the right choice when:

**✅ You're building an application, not a website**

If your project is primarily about **user interaction** rather than **content consumption**, React's strengths outweigh its costs.

**Application characteristics**:
- Users spend minutes to hours in a single session
- The UI updates frequently based on user actions
- State management is complex (shopping cart, multi-step forms, real-time collaboration)
- You need offline functionality or client-side data persistence
- The experience is fundamentally interactive (drag-and-drop, real-time updates)

**✅ You have complex state synchronization needs**

If your UI has multiple components that need to stay in sync, React's unidirectional data flow and state management tools provide real value.

**✅ You're building for a team**

If multiple developers will work on the codebase over years, React's component model and TypeScript integration provide structure and maintainability.

**✅ You need a rich ecosystem**

If you're building features that benefit from existing libraries (data tables, forms, charts, animations), React's ecosystem is unmatched.

**✅ Performance is acceptable with optimization**

If you can achieve acceptable performance with code splitting, lazy loading, and SSR/SSG (via Next.js), React's developer experience benefits are worth the baseline cost.

### The Honest Assessment: When React Doesn't Make Sense

React is the wrong choice when:

**❌ You're building content-focused sites**

If your project is primarily about **delivering content** (blogs, documentation, marketing sites, portfolios), React's JavaScript overhead hurts more than it helps.

**Content site characteristics**:
- Users read more than they interact
- Most pages are static or change infrequently
- SEO and initial load performance are critical
- Interactivity is minimal (maybe a contact form or search)
- You don't need client-side routing

**Better alternatives**: Astro, Hugo, Jekyll, Eleventy, or plain HTML/CSS with progressive enhancement.

**❌ Performance is non-negotiable**

If you're building for:
- Emerging markets with slow networks and low-end devices
- Users on metered connections
- Scenarios where every millisecond of load time matters (e-commerce conversion optimization)

React's baseline JavaScript cost is a fundamental limitation. Even with perfect optimization, you can't beat the performance of HTML that works without JavaScript.

**❌ You're a solo developer building a simple project**

If you're building a personal project or small business site, React's complexity is overhead you don't need. The time you spend configuring build tools and managing dependencies could be spent building features.

**❌ You need maximum longevity with minimal maintenance**

If you want to build something once and have it work for 10+ years with minimal updates, static HTML is far more durable than any JavaScript framework.

### React's Actual Position: A Powerful Tool with a Specific Domain

React is not "the best framework" or "the future of web development." It's a **sophisticated tool optimized for building complex, interactive applications**.

**The mental model**:

```
Problem Complexity
     ↑
     │                                    ┌─────────────┐
     │                                    │   React     │
     │                                    │  (Complex   │
     │                                    │   Apps)     │
     │                          ┌─────────┴─────────────┤
     │                          │   Astro/Eleventy      │
     │                          │  (Progressive         │
     │                          │   Enhancement)        │
     │                ┌─────────┴───────────────────────┤
     │                │   Static HTML/CSS               │
     │                │  (Content Sites)                │
     └────────────────┴─────────────────────────────────┴──→
                                              JavaScript Complexity
```

React sits in the upper-right quadrant: high problem complexity, high JavaScript complexity. It's the right tool when both are justified. It's the wrong tool when you're adding JavaScript complexity without corresponding problem complexity.

### The Professional Developer's Perspective

A professional React developer should be able to say:

> "I'm an expert in React, and for this project, I recommend we don't use React."

That's not a contradiction. It's wisdom. You understand React deeply enough to know its limitations. You care more about solving the user's problem than using your favorite tool.

**The decision framework**:

1. **Start with the user's needs**: What are they trying to accomplish?
2. **Identify the core complexity**: Is it content delivery or application logic?
3. **Choose the simplest tool that solves the problem**: Prefer less JavaScript, not more.
4. **Justify additional complexity**: Every framework, library, and abstraction must earn its place.

React has earned its place in the ecosystem by solving real problems for complex applications. But it hasn't earned a place in every project.

### Looking Forward: React's Evolution

React continues to evolve, and some of its recent developments address its historical weaknesses:

**Server Components** (React 18+, Next.js 13+):
- Reduce client-side JavaScript by rendering components on the server
- Blur the line between React and static site generators
- Make React more viable for content-heavy sites

**Streaming SSR**:
- Improve Time to First Byte (TTFB) and perceived performance
- Allow progressive rendering of complex pages

**Concurrent Rendering**:
- Better handling of expensive updates
- Improved user experience during heavy computation

These features make React more competitive with lighter-weight alternatives. But they also add complexity. The fundamental trade-off remains: React gives you power at the cost of complexity.

### The Ecosystem Reality

In 2025, the web development ecosystem is **pluralistic**. There's no single "best" approach:

- **Astro** is excellent for content sites with islands of interactivity
- **SvelteKit** offers a lighter-weight alternative to Next.js
- **Remix** provides a different take on server-side React
- **Solid.js** delivers React-like DX with better performance
- **HTMX** enables interactivity without a JavaScript framework
- **Qwik** optimizes for instant interactivity with resumability

React is one option among many. A good option for many use cases, but not the only option, and not always the best option.

### Your Responsibility as a React Developer

You've invested significant time learning React. That investment is valuable—React skills are in high demand and will remain relevant for years. But don't let that investment bias your technical decisions.

**Your responsibility**:

1. **Understand React's trade-offs deeply**: Know what you're paying for and what you're getting.
2. **Learn the alternatives**: Understand when other tools are better suited.
3. **Advocate for the user**: Choose tools based on user needs, not developer preferences.
4. **Stay humble**: The best framework is the one that solves the problem with the least complexity.

React is a powerful tool. Use it wisely.

## When to consider alternatives

## When to consider alternatives

Let's move from philosophy to practice. You're starting a new project. How do you decide whether React is the right choice? This section provides a systematic decision framework with concrete examples.

### The Decision Framework: Start with Constraints

Before evaluating frameworks, identify your project's constraints. These are non-negotiable requirements that eliminate certain options.

**Critical constraints to identify**:

1. **Performance budget**: What's the maximum acceptable Time to Interactive (TTI)?
2. **Target audience**: What devices and network conditions do they use?
3. **SEO requirements**: How critical is search engine visibility?
4. **Team expertise**: What does your team already know?
5. **Maintenance capacity**: How much ongoing maintenance can you sustain?
6. **Timeline**: How quickly do you need to ship?

Let's work through real scenarios to see how these constraints guide decisions.

### Scenario 1: Marketing Website for a SaaS Product

**Project requirements**:
- 5-10 pages (home, features, pricing, about, blog)
- Contact form and email signup
- Blog with 50+ articles
- Excellent SEO required (primary acquisition channel)
- Fast load times critical (conversion optimization)
- Small team (2 developers)
- Infrequent updates (monthly blog posts)

**Constraint analysis**:

```
Performance budget: < 1s TTI (conversion-critical)
Target audience: Global, mixed network conditions
SEO requirements: Critical (organic search is primary channel)
Team expertise: 2 developers, comfortable with HTML/CSS/JS
Maintenance capacity: Low (small team, other priorities)
Timeline: 4 weeks to launch
```

**React evaluation**:

❌ **Performance**: React's baseline ~100KB + hydration time exceeds budget
❌ **SEO**: Possible with Next.js SSG, but adds complexity
❌ **Maintenance**: React ecosystem churn requires ongoing updates
❌ **Complexity**: Build tooling, deployment, and optimization overhead
✅ **Team expertise**: Team could learn React, but it's not necessary

**Verdict**: React is the wrong choice.

**Better alternative**: **Astro** or **Hugo**

**Why Astro**:
- Zero JavaScript by default (perfect for content)
- Can add React components for interactive elements (contact form)
- Excellent SEO (static HTML)
- Fast builds and deploys
- Minimal maintenance (no framework updates)

**Implementation approach**:

```bash
# Create Astro project
npm create astro@latest saas-marketing

# Project structure
src/
├── pages/
│   ├── index.astro          # Home page (static)
│   ├── features.astro       # Features page (static)
│   ├── pricing.astro        # Pricing page (static)
│   └── blog/
│       └── [...slug].astro  # Blog posts (static)
├── components/
│   ├── ContactForm.tsx      # React component (interactive)
│   └── EmailSignup.tsx      # React component (interactive)
└── layouts/
    └── Layout.astro         # Shared layout
```

**Contact form** (the only interactive element):

```tsx
// src/components/ContactForm.tsx
import { useState } from 'react';

export default function ContactForm() {
  const [status, setStatus] = useState<'idle' | 'submitting' | 'success' | 'error'>('idle');

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setStatus('submitting');
    
    const formData = new FormData(e.currentTarget);
    
    try {
      const response = await fetch('/api/contact', {
        method: 'POST',
        body: formData,
      });
      
      if (response.ok) {
        setStatus('success');
      } else {
        setStatus('error');
      }
    } catch (error) {
      setStatus('error');
    }
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <input
        type="email"
        name="email"
        placeholder="Your email"
        required
        className="w-full px-4 py-2 border rounded"
      />
      <textarea
        name="message"
        placeholder="Your message"
        required
        className="w-full px-4 py-2 border rounded"
      />
      <button
        type="submit"
        disabled={status === 'submitting'}
        className="px-6 py-2 bg-blue-600 text-white rounded"
      >
        {status === 'submitting' ? 'Sending...' : 'Send Message'}
      </button>
      {status === 'success' && (
        <p className="text-green-600">Message sent successfully!</p>
      )}
      {status === 'error' && (
        <p className="text-red-600">Failed to send message. Please try again.</p>
      )}
    </form>
  );
}
```

**Using the React component in Astro**:

```astro
---
// src/pages/contact.astro
import Layout from '../layouts/Layout.astro';
import ContactForm from '../components/ContactForm';
---

<Layout title="Contact Us">
  <h1>Get in Touch</h1>
  <p>We'd love to hear from you.</p>
  
  <!-- Only this component loads JavaScript -->
  <ContactForm client:load />
</Layout>
```

**Result**:
- **Home page**: 0 KB JavaScript, instant interactivity
- **Blog pages**: 0 KB JavaScript, instant interactivity
- **Contact page**: ~15 KB JavaScript (only the form component)
- **TTI**: < 500ms on 3G
- **SEO**: Perfect (static HTML)
- **Maintenance**: Minimal (Astro is stable, no framework churn)

**Cost comparison**:

```
React (Next.js) approach:
- Initial bundle: 100+ KB
- TTI: 1-2s on 3G
- Build complexity: High
- Maintenance: Ongoing framework updates

Astro approach:
- Initial bundle: 0 KB (15 KB on contact page only)
- TTI: < 500ms
- Build complexity: Low
- Maintenance: Minimal
```

The Astro approach delivers better performance, better SEO, and lower maintenance—all while still using React for the one component that needs interactivity.

### Scenario 2: Internal Admin Dashboard

**Project requirements**:
- Complex data tables with sorting, filtering, pagination
- Real-time updates from WebSocket
- Multi-step forms with validation
- Role-based access control
- 20+ screens
- Used by 50 internal employees (all on modern browsers, good network)
- Frequent feature additions

**Constraint analysis**:

```
Performance budget: Flexible (internal tool, good network)
Target audience: Internal employees, modern browsers
SEO requirements: None (behind authentication)
Team expertise: 3 developers, experienced with React
Maintenance capacity: High (active development)
Timeline: 3 months to MVP, ongoing development
```

**React evaluation**:

✅ **Complexity**: Dashboard has complex state management needs
✅ **Interactivity**: Highly interactive, not content-focused
✅ **Team expertise**: Team is already proficient in React
✅ **Ecosystem**: Can leverage React Query, TanStack Table, React Hook Form
✅ **Performance**: Acceptable for internal tool with good network
✅ **Maintenance**: Active development justifies framework investment

**Verdict**: React is the right choice.

**Recommended stack**: **Next.js + React Query + Zustand + TanStack Table**

**Why this stack**:
- **Next.js**: API routes, authentication, file-based routing
- **React Query**: Server state management, real-time updates
- **Zustand**: Global client state (user preferences, UI state)
- **TanStack Table**: Powerful, flexible data tables
- **React Hook Form + Zod**: Type-safe form handling

**Implementation approach**:

```bash
# Project structure
src/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   ├── (dashboard)/
│   │   ├── users/
│   │   │   ├── page.tsx          # User list with table
│   │   │   └── [id]/
│   │   │       └── page.tsx      # User detail
│   │   ├── products/
│   │   │   └── page.tsx
│   │   └── layout.tsx            # Dashboard layout with nav
│   └── api/
│       ├── users/
│       │   └── route.ts
│       └── auth/
│           └── [...nextauth]/
│               └── route.ts
├── components/
│   ├── tables/
│   │   └── UserTable.tsx
│   ├── forms/
│   │   └── UserForm.tsx
│   └── ui/                       # shadcn/ui components
└── lib/
    ├── api.ts                    # API client
    ├── auth.ts                   # NextAuth config
    └── store.ts                  # Zustand store
```

**User table with TanStack Table**:

```tsx
// src/components/tables/UserTable.tsx
'use client';

import { useQuery } from '@tanstack/react-query';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getFilteredRowModel,
  getPaginationRowModel,
  flexRender,
} from '@tanstack/react-table';
import { useState } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
  role: string;
  status: 'active' | 'inactive';
}

export default function UserTable() {
  const [sorting, setSorting] = useState([]);
  const [filtering, setFiltering] = useState('');
  const [pagination, setPagination] = useState({
    pageIndex: 0,
    pageSize: 10,
  });

  // Fetch users with React Query
  const { data, isLoading, error } = useQuery({
    queryKey: ['users', pagination, sorting, filtering],
    queryFn: async () => {
      const params = new URLSearchParams({
        page: pagination.pageIndex.toString(),
        pageSize: pagination.pageSize.toString(),
        sort: JSON.stringify(sorting),
        filter: filtering,
      });
      const response = await fetch(`/api/users?${params}`);
      if (!response.ok) throw new Error('Failed to fetch users');
      return response.json();
    },
  });

  const columns = [
    {
      accessorKey: 'name',
      header: 'Name',
    },
    {
      accessorKey: 'email',
      header: 'Email',
    },
    {
      accessorKey: 'role',
      header: 'Role',
    },
    {
      accessorKey: 'status',
      header: 'Status',
      cell: ({ getValue }) => {
        const status = getValue() as string;
        return (
          <span
            className={`px-2 py-1 rounded text-sm ${
              status === 'active'
                ? 'bg-green-100 text-green-800'
                : 'bg-gray-100 text-gray-800'
            }`}
          >
            {status}
          </span>
        );
      },
    },
  ];

  const table = useReactTable({
    data: data?.users ?? [],
    columns,
    state: {
      sorting,
      globalFilter: filtering,
      pagination,
    },
    onSortingChange: setSorting,
    onGlobalFilterChange: setFiltering,
    onPaginationChange: setPagination,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    manualPagination: true,
    pageCount: data?.pageCount ?? 0,
  });

  if (isLoading) return <div>Loading users...</div>;
  if (error) return <div>Error loading users</div>;

  return (
    <div className="space-y-4">
      <input
        type="text"
        value={filtering}
        onChange={(e) => setFiltering(e.target.value)}
        placeholder="Search users..."
        className="px-4 py-2 border rounded"
      />

      <table className="w-full border-collapse">
        <thead>
          {table.getHeaderGroups().map((headerGroup) => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map((header) => (
                <th
                  key={header.id}
                  onClick={header.column.getToggleSortingHandler()}
                  className="px-4 py-2 text-left border-b cursor-pointer hover:bg-gray-50"
                >
                  {flexRender(
                    header.column.columnDef.header,
                    header.getContext()
                  )}
                  {header.column.getIsSorted() && (
                    <span className="ml-2">
                      {header.column.getIsSorted() === 'asc' ? '↑' : '↓'}
                    </span>
                  )}
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {table.getRowModel().rows.map((row) => (
            <tr key={row.id} className="hover:bg-gray-50">
              {row.getVisibleCells().map((cell) => (
                <td key={cell.id} className="px-4 py-2 border-b">
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>

      <div className="flex items-center justify-between">
        <div>
          Showing {table.getState().pagination.pageIndex * table.getState().pagination.pageSize + 1} to{' '}
          {Math.min(
            (table.getState().pagination.pageIndex + 1) * table.getState().pagination.pageSize,
            data?.totalCount ?? 0
          )}{' '}
          of {data?.totalCount ?? 0} users
        </div>
        <div className="flex gap-2">
          <button
            onClick={() => table.previousPage()}
            disabled={!table.getCanPreviousPage()}
            className="px-4 py-2 border rounded disabled:opacity-50"
          >
            Previous
          </button>
          <button
            onClick={() => table.nextPage()}
            disabled={!table.getCanNextPage()}
            className="px-4 py-2 border rounded disabled:opacity-50"
          >
            Next
          </button>
        </div>
      </div>
    </div>
  );
}
```

**Result**:
- **Complex state management**: React Query handles server state, Zustand handles client state
- **Rich interactivity**: Sorting, filtering, pagination all work smoothly
- **Type safety**: TypeScript catches errors at compile time
- **Developer experience**: Hot reload, React DevTools, excellent debugging
- **Maintainability**: Clear component structure, reusable patterns
- **Performance**: Acceptable for internal tool (100-200ms interactions)

**Why React was the right choice here**:
1. **Complexity justifies overhead**: The dashboard has genuine complexity that benefits from React's state management
2. **Team expertise**: Team is already proficient, no learning curve
3. **Ecosystem value**: Libraries like TanStack Table and React Query save weeks of development
4. **Performance acceptable**: Internal tool with good network conditions
5. **Active development**: Ongoing feature work justifies framework investment

### Scenario 3: E-commerce Product Catalog

**Project requirements**:
- 10,000+ products
- Product listing with filters, sorting, search
- Product detail pages
- Shopping cart
- Checkout flow
- Excellent SEO required (organic search is primary channel)
- Fast load times critical (conversion optimization)
- Global audience (mixed network conditions)

**Constraint analysis**:

```
Performance budget: < 2s TTI (conversion-critical)
Target audience: Global, mixed network conditions
SEO requirements: Critical (organic search is primary channel)
Team expertise: 2 developers, comfortable with React
Maintenance capacity: Medium (small team, but e-commerce is core business)
Timeline: 8 weeks to launch
```

**React evaluation**:

⚠️ **Performance**: React's baseline cost is concerning for conversion optimization
✅ **SEO**: Next.js SSG/ISR can handle this
✅ **Interactivity**: Shopping cart and filters need client-side state
✅ **Complexity**: Product catalog has moderate complexity
⚠️ **Maintenance**: E-commerce requires ongoing optimization

**Verdict**: React (Next.js) is viable, but requires careful optimization.

**Recommended approach**: **Next.js with aggressive optimization**

**Why Next.js**:
- **SSG for product pages**: Pre-render all product pages at build time
- **ISR for catalog**: Incremental Static Regeneration for product listings
- **Server Components**: Minimize client-side JavaScript
- **Image optimization**: next/image for product photos
- **Code splitting**: Route-based splitting to minimize initial bundle

**Critical optimization strategy**:

```typescript
// src/app/products/[slug]/page.tsx
import { Suspense } from 'react';
import Image from 'next/image';
import { notFound } from 'next/navigation';
import AddToCartButton from '@/components/AddToCartButton';
import ProductReviews from '@/components/ProductReviews';

interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  images: string[];
  slug: string;
}

// Generate static paths for all products at build time
export async function generateStaticParams() {
  const products = await fetch('https://api.example.com/products').then(res => res.json());
  
  return products.map((product: Product) => ({
    slug: product.slug,
  }));
}

// Fetch product data at build time
async function getProduct(slug: string): Promise<Product | null> {
  const response = await fetch(`https://api.example.com/products/${slug}`, {
    next: { revalidate: 3600 }, // Revalidate every hour
  });
  
  if (!response.ok) return null;
  return response.json();
}

// Server Component (no JavaScript sent to client)
export default async function ProductPage({
  params,
}: {
  params: { slug: string };
}) {
  const product = await getProduct(params.slug);
  
  if (!product) {
    notFound();
  }

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
        {/* Product images - Server Component */}
        <div>
          <Image
            src={product.images[0]}
            alt={product.name}
            width={600}
            height={600}
            priority
            className="w-full rounded-lg"
          />
        </div>

        {/* Product info - Server Component */}
        <div>
          <h1 className="text-3xl font-bold mb-4">{product.name}</h1>
          <p className="text-2xl font-semibold mb-4">${product.price}</p>
          <p className="text-gray-700 mb-6">{product.description}</p>

          {/* Only this button is a Client Component */}
          <AddToCartButton productId={product.id} />
        </div>
      </div>

      {/* Reviews loaded separately to avoid blocking */}
      <Suspense fallback={<div>Loading reviews...</div>}>
        <ProductReviews productId={product.id} />
      </Suspense>
    </div>
  );
}
```

**Add to cart button** (the only Client Component on the page):

```tsx
// src/components/AddToCartButton.tsx
'use client';

import { useCart } from '@/lib/cart-store';
import { useState } from 'react';

interface AddToCartButtonProps {
  productId: string;
}

export default function AddToCartButton({ productId }: AddToCartButtonProps) {
  const [quantity, setQuantity] = useState(1);
  const addToCart = useCart((state) => state.addItem);
  const [isAdding, setIsAdding] = useState(false);

  async function handleAddToCart() {
    setIsAdding(true);
    await addToCart(productId, quantity);
    setIsAdding(false);
  }

  return (
    <div className="flex items-center gap-4">
      <input
        type="number"
        min="1"
        value={quantity}
        onChange={(e) => setQuantity(parseInt(e.target.value))}
        className="w-20 px-3 py-2 border rounded"
      />
      <button
        onClick={handleAddToCart}
        disabled={isAdding}
        className="px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:opacity-50"
      >
        {isAdding ? 'Adding...' : 'Add to Cart'}
      </button>
    </div>
  );
}
```

**Result**:
- **Product page bundle**: ~20 KB JavaScript (only the cart button)
- **TTI**: < 1s on 3G (most content is static HTML)
- **SEO**: Perfect (static HTML with all content)
- **Conversion optimization**: Fast load times, instant interactivity
- **Maintenance**: Moderate (Next.js is stable, but requires performance monitoring)

**Performance comparison**:

```
Full client-side React approach:
- Initial bundle: 150+ KB
- TTI: 2-3s on 3G
- SEO: Requires SSR, adds complexity
- Conversion rate: Baseline

Next.js with Server Components:
- Initial bundle: 20 KB
- TTI: < 1s on 3G
- SEO: Perfect (static HTML)
- Conversion rate: +15-25% (industry data for 1s faster load)
```

**Why Next.js was the right choice here**:
1. **Hybrid approach**: Server Components for content, Client Components for interactivity
2. **SEO requirements met**: Static HTML for all product pages
3. **Performance acceptable**: Aggressive optimization achieves conversion-critical performance
4. **Shopping cart complexity**: Client-side state management justified for cart functionality
5. **Image optimization**: next/image provides automatic optimization for product photos

### Scenario 4: Documentation Site

**Project requirements**:
- 200+ documentation pages
- Search functionality
- Code syntax highlighting
- Version selector
- Minimal interactivity (mostly reading)
- Excellent SEO required
- Fast load times critical
- Open source project (community contributions)

**Constraint analysis**:

```
Performance budget: < 1s TTI
Target audience: Developers, global, mixed network conditions
SEO requirements: Critical (primary discovery method)
Team expertise: Open source contributors, varied skill levels
Maintenance capacity: Low (volunteer maintainers)
Timeline: Ongoing (documentation evolves with project)
```

**React evaluation**:

❌ **Performance**: React's JavaScript overhead hurts reading experience
❌ **Complexity**: Build tooling creates barrier for contributors
❌ **Maintenance**: Framework updates burden volunteer maintainers
⚠️ **Search**: Could use client-side search, but server-side is simpler
✅ **Syntax highlighting**: Works with any framework

**Verdict**: React is the wrong choice.

**Better alternative**: **Astro** or **Docusaurus** (if you must use React)

**Why Astro**:
- Zero JavaScript by default (perfect for documentation)
- Markdown-based content (easy for contributors)
- Built-in syntax highlighting
- Fast builds
- Minimal maintenance

**Why Docusaurus** (if React is required):
- Built specifically for documentation
- Optimized for performance (despite using React)
- Excellent search (Algolia integration)
- Versioning built-in
- Large community (Facebook-backed)

**Implementation approach** (Astro):

```bash
# Create Astro docs site
npm create astro@latest docs-site -- --template docs

# Project structure
src/
├── content/
│   └── docs/
│       ├── getting-started/
│       │   ├── installation.md
│       │   └── quick-start.md
│       ├── guides/
│       │   └── advanced-usage.md
│       └── api/
│           └── reference.md
├── components/
│   ├── Search.tsx          # Only interactive component
│   └── VersionSelector.tsx # Only interactive component
└── pages/
    └── [...slug].astro     # Dynamic doc pages
```

**Documentation page** (static):

```astro
---
// src/pages/[...slug].astro
import { getCollection } from 'astro:content';
import Layout from '../layouts/Layout.astro';
import Search from '../components/Search';

export async function getStaticPaths() {
  const docs = await getCollection('docs');
  return docs.map(doc => ({
    params: { slug: doc.slug },
    props: { doc },
  }));
}

const { doc } = Astro.props;
const { Content } = await doc.render();
---

<Layout title={doc.data.title}>
  <aside>
    <!-- Static navigation -->
    <nav>
      <!-- Table of contents -->
    </nav>
  </aside>

  <main>
    <h1>{doc.data.title}</h1>
    
    <!-- Markdown content rendered to HTML -->
    <Content />
  </main>

  <!-- Only search is interactive -->
  <Search client:load />
</Layout>
```

**Search component** (the only JavaScript):

```tsx
// src/components/Search.tsx
import { useState, useEffect } from 'react';
import Fuse from 'fuse.js';

interface SearchResult {
  title: string;
  url: string;
  excerpt: string;
}

export default function Search() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const [fuse, setFuse] = useState<Fuse<SearchResult> | null>(null);

  useEffect(() => {
    // Load search index
    fetch('/search-index.json')
      .then(res => res.json())
      .then(data => {
        const fuseInstance = new Fuse(data, {
          keys: ['title', 'content'],
          threshold: 0.3,
        });
        setFuse(fuseInstance);
      });
  }, []);

  useEffect(() => {
    if (!fuse || !query) {
      setResults([]);
      return;
    }

    const searchResults = fuse.search(query).slice(0, 10);
    setResults(searchResults.map(result => result.item));
  }, [query, fuse]);

  return (
    <div className="search">
      <input
        type="search"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search documentation..."
        className="w-full px-4 py-2 border rounded"
      />
      {results.length > 0 && (
        <ul className="mt-2 border rounded bg-white shadow-lg">
          {results.map((result) => (
            <li key={result.url}>
              <a href={result.url} className="block px-4 py-2 hover:bg-gray-100">
                <div className="font-semibold">{result.title}</div>
                <div className="text-sm text-gray-600">{result.excerpt}</div>
              </a>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

**Result**:
- **Documentation pages**: 0 KB JavaScript (pure HTML)
- **Search page**: ~30 KB JavaScript (only when search is used)
- **TTI**: < 300ms (instant for reading)
- **SEO**: Perfect (static HTML)
- **Contributor experience**: Write Markdown, no build complexity
- **Maintenance**: Minimal (Astro is stable)

**Why Astro was the right choice here**:
1. **Content-focused**: Documentation is 99% reading, 1% interaction
2. **Performance**: Zero JavaScript for reading experience
3. **Contributor-friendly**: Markdown is accessible to all skill levels
4. **Maintenance**: Minimal framework updates needed
5. **SEO**: Static HTML ensures perfect search engine visibility

### Decision Matrix: Quick Reference

Use this matrix to quickly evaluate whether React is appropriate for your project:

| Project Type | React? | Better Alternative | Why |
|--------------|--------|-------------------|-----|
| Marketing site | ❌ | Astro, Hugo | Content-focused, SEO-critical, minimal interactivity |
| Blog | ❌ | Astro, Hugo, Jekyll | Content-focused, SEO-critical, minimal interactivity |
| Documentation | ❌ | Astro, Docusaurus | Content-focused, contributor-friendly, minimal interactivity |
| Portfolio | ❌ | Astro, plain HTML | Content-focused, minimal interactivity |
| Landing page | ❌ | Astro, plain HTML | Performance-critical, minimal interactivity |
| E-commerce (content-heavy) | ⚠️ | Next.js (optimized) | Hybrid: content + interactivity, requires optimization |
| E-commerce (app-like) | ✅ | Next.js, React | Complex state (cart, checkout), interactivity-focused |
| Admin dashboard | ✅ | Next.js, React | Complex state, highly interactive, internal tool |
| SaaS application | ✅ | Next.js, React | Complex state, highly interactive, app-like |
| Social media app | ✅ | Next.js, React | Real-time updates, complex state, highly interactive |
| Collaborative tool | ✅ | Next.js, React | Real-time collaboration, complex state, highly interactive |
| Data visualization | ✅ | React | Complex interactions, dynamic updates |
| Form-heavy app | ✅ | React | Complex validation, multi-step flows |

### The Litmus Test: Three Questions

When in doubt, ask these three questions:

**1. Is this primarily about content or interaction?**
- **Content**: Users read, browse, consume information → Consider alternatives
- **Interaction**: Users manipulate data, collaborate, perform tasks → React is viable

**2. What's the JavaScript budget?**
- **< 50 KB**: React is too heavy → Use Astro or plain HTML
- **50-150 KB**: React is viable with optimization → Use Next.js with Server Components
- **> 150 KB**: React is fine → Use React/Next.js freely

**3. What's the maintenance capacity?**
- **Low** (solo dev, side project): Avoid React's complexity → Use simpler tools
- **Medium** (small team, occasional updates): React is viable if justified → Evaluate carefully
- **High** (dedicated team, active development): React is fine → Use React/Next.js

If you answer "content", "< 50 KB", or "low" to any question, seriously consider alternatives to React.

### The Professional Approach: Justify Your Choice

Whatever you choose, be able to articulate why. A professional decision includes:

1. **Constraint analysis**: What are the non-negotiable requirements?
2. **Trade-off evaluation**: What are you optimizing for? What are you sacrificing?
3. **Alternative consideration**: What other options did you evaluate?
4. **Risk assessment**: What could go wrong? How will you mitigate it?
5. **Success criteria**: How will you measure whether the choice was correct?

**Example justification** (for choosing React):

> "We chose React for this project because:
> 1. The application is highly interactive (real-time collaboration)
> 2. Complex state management justifies React's overhead
> 3. Our team is already proficient in React
> 4. Performance is acceptable for our target audience (internal tool, good network)
> 5. We can leverage React Query and Zustand to accelerate development
> 
> We considered Svelte for better performance, but the ecosystem gap and team learning curve outweighed the benefits. We'll monitor bundle size and TTI to ensure performance remains acceptable."

**Example justification** (for choosing Astro over React):

> "We chose Astro for this project because:
> 1. The site is content-focused (blog + documentation)
> 2. SEO and performance are critical (organic search is primary channel)
> 3. Interactivity is minimal (contact form only)
> 4. React's JavaScript overhead would hurt conversion rates
> 5. Astro allows us to use React for the one interactive component
> 
> We considered Next.js with Server Components, but even optimized, it would add 50+ KB of JavaScript for no user benefit. We'll use Astro's React integration for the contact form to get the best of both worlds."

The ability to justify your choice—and to choose against React when appropriate—is the mark of a mature developer.

## Building maintainable applications that outlast framework churn

## Building maintainable applications that outlast framework churn

You've learned React. You've learned when to use it and when not to. Now let's address the elephant in the room: **frameworks change, but your application needs to keep working**.

In 2015, developers built React apps with class components, Redux, and Create React App. In 2025, those same apps need refactoring to use function components, React Query, and Vite or Next.js. The code still works, but it's "legacy" after just 10 years.

How do you build applications that survive framework churn? How do you write code that's maintainable not just today, but 5-10 years from now?

### The Uncomfortable Truth About Framework Longevity

Let's start with reality: **no JavaScript framework is permanent**.

**Framework lifecycle patterns**:

```
Year 0-2: Hype phase
- Rapid adoption
- Breaking changes common
- Ecosystem immature
- "This will change everything!"

Year 3-5: Maturity phase
- Stable API
- Rich ecosystem
- Best practices emerge
- "This is the standard"

Year 6-10: Maintenance phase
- Fewer breaking changes
- Ecosystem consolidation
- New frameworks emerge
- "This is legacy"

Year 10+: Legacy phase
- Security updates only
- Migration pressure
- Hiring challenges
- "We need to rewrite"
```

React is currently in the **maturity phase** (launched 2013, 12 years old). It's stable, widely adopted, and will remain relevant for years. But it won't last forever.

**Historical examples**:

- **jQuery** (2006): Dominated for a decade, now considered legacy
- **Backbone.js** (2010): Popular in early 2010s, now rarely used
- **Angular.js** (2010): Massive adoption, then Angular 2+ broke compatibility
- **Ember.js** (2011): Once a React competitor, now niche

React will eventually follow this pattern. Maybe in 5 years, maybe in 15. But it will happen.

### The Core Principle: Separate Business Logic from Framework Code

The key to longevity is **minimizing framework coupling**. The more your business logic depends on React-specific patterns, the harder it will be to migrate when the time comes.

**The architecture principle**:

```
┌─────────────────────────────────────────────────────────┐
│                    Your Application                      │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │         Business Logic (Framework-agnostic)      │   │
│  │  - Domain models                                 │   │
│  │  - Business rules                                │   │
│  │  - Data transformations                          │   │
│  │  - Validation logic                              │   │
│  └─────────────────────────────────────────────────┘   │
│                          ↑                               │
│                          │                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │         Adapter Layer (Framework-aware)          │   │
│  │  - API clients                                   │   │
│  │  - State management                              │   │
│  │  - Routing logic                                 │   │
│  └─────────────────────────────────────────────────┘   │
│                          ↑                               │
│                          │                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │         UI Layer (React-specific)                │   │
│  │  - Components                                    │   │
│  │  - Hooks                                         │   │
│  │  - Event handlers                                │   │
│  └─────────────────────────────────────────────────┘   │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**The goal**: Keep the top layer (business logic) as large as possible and completely framework-agnostic. When you eventually migrate from React to the next framework, you rewrite only the bottom two layers.

### Reference Implementation: E-commerce Order Processing

Let's build a realistic example that demonstrates maintainable architecture. We'll build an order processing system that could survive a framework migration.

**Project structure**:

```bash
src/
├── domain/                    # Framework-agnostic business logic
│   ├── models/
│   │   ├── Order.ts
│   │   ├── Product.ts
│   │   └── Customer.ts
│   ├── services/
│   │   ├── OrderService.ts
│   │   ├── PricingService.ts
│   │   └── InventoryService.ts
│   └── validation/
│       ├── orderValidation.ts
│       └── paymentValidation.ts
├── adapters/                  # Framework-aware adapters
│   ├── api/
│   │   ├── OrderApiClient.ts
│   │   └── ProductApiClient.ts
│   ├── storage/
│   │   └── LocalStorageAdapter.ts
│   └── state/
│       └── orderStore.ts
└── ui/                        # React-specific UI
    ├── components/
    │   ├── OrderForm.tsx
    │   ├── OrderSummary.tsx
    │   └── PaymentForm.tsx
    └── hooks/
        ├── useOrder.ts
        └── useProducts.ts
```

### Phase 1: Framework-Agnostic Business Logic

Start with pure TypeScript that has zero React dependencies. This code should work in Node.js, Deno, or any JavaScript environment.

**Domain model** (pure TypeScript):

```typescript
// src/domain/models/Order.ts

export interface OrderItem {
  productId: string;
  quantity: number;
  priceAtPurchase: number;
}

export interface ShippingAddress {
  street: string;
  city: string;
  state: string;
  zipCode: string;
  country: string;
}

export interface Order {
  id: string;
  customerId: string;
  items: OrderItem[];
  shippingAddress: ShippingAddress;
  status: 'draft' | 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';
  subtotal: number;
  tax: number;
  shipping: number;
  total: number;
  createdAt: Date;
  updatedAt: Date;
}

export type OrderDraft = Omit<Order, 'id' | 'status' | 'createdAt' | 'updatedAt'>;
```

**Business logic** (pure functions, no React):

```typescript
// src/domain/services/PricingService.ts

import type { OrderItem, ShippingAddress } from '../models/Order';

export class PricingService {
  private readonly TAX_RATE = 0.08; // 8% tax
  private readonly FREE_SHIPPING_THRESHOLD = 50;
  private readonly STANDARD_SHIPPING_COST = 5.99;

  calculateSubtotal(items: OrderItem[]): number {
    return items.reduce((sum, item) => {
      return sum + item.priceAtPurchase * item.quantity;
    }, 0);
  }

  calculateTax(subtotal: number): number {
    return subtotal * this.TAX_RATE;
  }

  calculateShipping(subtotal: number, address: ShippingAddress): number {
    // Free shipping over threshold
    if (subtotal >= this.FREE_SHIPPING_THRESHOLD) {
      return 0;
    }

    // International shipping costs more
    if (address.country !== 'US') {
      return this.STANDARD_SHIPPING_COST * 2;
    }

    return this.STANDARD_SHIPPING_COST;
  }

  calculateTotal(
    items: OrderItem[],
    address: ShippingAddress
  ): {
    subtotal: number;
    tax: number;
    shipping: number;
    total: number;
  } {
    const subtotal = this.calculateSubtotal(items);
    const tax = this.calculateTax(subtotal);
    const shipping = this.calculateShipping(subtotal, address);
    const total = subtotal + tax + shipping;

    return { subtotal, tax, shipping, total };
  }
}
```

**Validation logic** (pure functions):

```typescript
// src/domain/validation/orderValidation.ts

import type { OrderDraft } from '../models/Order';

export interface ValidationError {
  field: string;
  message: string;
}

export class OrderValidator {
  validate(order: OrderDraft): ValidationError[] {
    const errors: ValidationError[] = [];

    // Validate items
    if (order.items.length === 0) {
      errors.push({
        field: 'items',
        message: 'Order must contain at least one item',
      });
    }

    for (const item of order.items) {
      if (item.quantity <= 0) {
        errors.push({
          field: `items.${item.productId}.quantity`,
          message: 'Quantity must be greater than 0',
        });
      }

      if (item.priceAtPurchase < 0) {
        errors.push({
          field: `items.${item.productId}.price`,
          message: 'Price cannot be negative',
        });
      }
    }

    // Validate shipping address
    if (!order.shippingAddress.street) {
      errors.push({
        field: 'shippingAddress.street',
        message: 'Street address is required',
      });
    }

    if (!order.shippingAddress.city) {
      errors.push({
        field: 'shippingAddress.city',
        message: 'City is required',
      });
    }

    if (!order.shippingAddress.zipCode) {
      errors.push({
        field: 'shippingAddress.zipCode',
        message: 'ZIP code is required',
      });
    }

    // Validate ZIP code format (US only for simplicity)
    if (order.shippingAddress.country === 'US') {
      const zipRegex = /^\d{5}(-\d{4})?$/;
      if (!zipRegex.test(order.shippingAddress.zipCode)) {
        errors.push({
          field: 'shippingAddress.zipCode',
          message: 'Invalid ZIP code format',
        });
      }
    }

    return errors;
  }

  isValid(order: OrderDraft): boolean {
    return this.validate(order).length === 0;
  }
}
```

**Why this matters**:

Notice that none of this code imports React, uses hooks, or depends on any framework. You could:
- Run it in Node.js for server-side validation
- Use it in a CLI tool for batch processing
- Test it without any React testing utilities
- Migrate it to Vue, Svelte, or Angular with zero changes

This is **framework-agnostic business logic**. It will outlive React.

### Phase 2: Framework-Aware Adapters

Now we create adapters that connect our business logic to the outside world. These adapters are framework-aware but keep React-specific code minimal.

**API client** (framework-aware, but not React-specific):

```typescript
// src/adapters/api/OrderApiClient.ts

import type { Order, OrderDraft } from '@/domain/models/Order';

export class OrderApiClient {
  private baseUrl: string;

  constructor(baseUrl: string = '/api') {
    this.baseUrl = baseUrl;
  }

  async createOrder(draft: OrderDraft): Promise<Order> {
    const response = await fetch(`${this.baseUrl}/orders`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(draft),
    });

    if (!response.ok) {
      throw new Error(`Failed to create order: ${response.statusText}`);
    }

    return response.json();
  }

  async getOrder(orderId: string): Promise<Order> {
    const response = await fetch(`${this.baseUrl}/orders/${orderId}`);

    if (!response.ok) {
      throw new Error(`Failed to fetch order: ${response.statusText}`);
    }

    return response.json();
  }

  async updateOrder(orderId: string, updates: Partial<Order>): Promise<Order> {
    const response = await fetch(`${this.baseUrl}/orders/${orderId}`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(updates),
    });

    if (!response.ok) {
      throw new Error(`Failed to update order: ${response.statusText}`);
    }

    return response.json();
  }

  async cancelOrder(orderId: string): Promise<Order> {
    return this.updateOrder(orderId, { status: 'cancelled' });
  }
}
```

**State management** (Zustand, but could be replaced):

```typescript
// src/adapters/state/orderStore.ts

import { create } from 'zustand';
import type { OrderItem, ShippingAddress } from '@/domain/models/Order';
import { PricingService } from '@/domain/services/PricingService';
import { OrderValidator } from '@/domain/validation/orderValidation';

interface OrderState {
  items: OrderItem[];
  shippingAddress: ShippingAddress | null;
  
  // Computed values
  subtotal: number;
  tax: number;
  shipping: number;
  total: number;
  
  // Actions
  addItem: (productId: string, quantity: number, price: number) => void;
  removeItem: (productId: string) => void;
  updateQuantity: (productId: string, quantity: number) => void;
  setShippingAddress: (address: ShippingAddress) => void;
  clearOrder: () => void;
  
  // Validation
  validate: () => string[];
}

const pricingService = new PricingService();
const validator = new OrderValidator();

export const useOrderStore = create<OrderState>((set, get) => ({
  items: [],
  shippingAddress: null,
  subtotal: 0,
  tax: 0,
  shipping: 0,
  total: 0,

  addItem: (productId, quantity, price) => {
    set((state) => {
      const existingItem = state.items.find(item => item.productId === productId);
      
      let newItems: OrderItem[];
      if (existingItem) {
        newItems = state.items.map(item =>
          item.productId === productId
            ? { ...item, quantity: item.quantity + quantity }
            : item
        );
      } else {
        newItems = [
          ...state.items,
          { productId, quantity, priceAtPurchase: price },
        ];
      }

      const pricing = state.shippingAddress
        ? pricingService.calculateTotal(newItems, state.shippingAddress)
        : { subtotal: 0, tax: 0, shipping: 0, total: 0 };

      return {
        items: newItems,
        ...pricing,
      };
    });
  },

  removeItem: (productId) => {
    set((state) => {
      const newItems = state.items.filter(item => item.productId !== productId);
      
      const pricing = state.shippingAddress
        ? pricingService.calculateTotal(newItems, state.shippingAddress)
        : { subtotal: 0, tax: 0, shipping: 0, total: 0 };

      return {
        items: newItems,
        ...pricing,
      };
    });
  },

  updateQuantity: (productId, quantity) => {
    set((state) => {
      const newItems = state.items.map(item =>
        item.productId === productId
          ? { ...item, quantity }
          : item
      );

      const pricing = state.shippingAddress
        ? pricingService.calculateTotal(newItems, state.shippingAddress)
        : { subtotal: 0, tax: 0, shipping: 0, total: 0 };

      return {
        items: newItems,
        ...pricing,
      };
    });
  },

  setShippingAddress: (address) => {
    set((state) => {
      const pricing = pricingService.calculateTotal(state.items, address);

      return {
        shippingAddress: address,
        ...pricing,
      };
    });
  },

  clearOrder: () => {
    set({
      items: [],
      shippingAddress: null,
      subtotal: 0,
      tax: 0,
      shipping: 0,
      total: 0,
    });
  },

  validate: () => {
    const state = get();
    
    if (!state.shippingAddress) {
      return ['Shipping address is required'];
    }

    const errors = validator.validate({
      customerId: '', // Would come from auth context
      items: state.items,
      shippingAddress: state.shippingAddress,
      subtotal: state.subtotal,
      tax: state.tax,
      shipping: state.shipping,
      total: state.total,
    });

    return errors.map(error => error.message);
  },
}));
```

**Why this matters**:

The adapter layer uses Zustand (a React state management library), but notice:
- All business logic is delegated to `PricingService` and `OrderValidator`
- The store is just glue code connecting UI to business logic
- If you migrate from Zustand to Redux or Jotai, you only rewrite this file
- The business logic remains untouched

### Phase 3: React UI Layer

Finally, we build React components that use the adapters. This is the only layer that's truly React-specific.

**Order form component**:

```tsx
// src/ui/components/OrderForm.tsx
'use client';

import { useState } from 'react';
import { useOrderStore } from '@/adapters/state/orderStore';
import { OrderApiClient } from '@/adapters/api/OrderApiClient';

const apiClient = new OrderApiClient();

export default function OrderForm() {
  const {
    items,
    shippingAddress,
    subtotal,
    tax,
    shipping,
    total,
    setShippingAddress,
    validate,
    clearOrder,
  } = useOrderStore();

  const [formData, setFormData] = useState({
    street: '',
    city: '',
    state: '',
    zipCode: '',
    country: 'US',
  });

  const [errors, setErrors] = useState<string[]>([]);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [orderConfirmed, setOrderConfirmed] = useState(false);

  function handleInputChange(e: React.ChangeEvent<HTMLInputElement | HTMLSelectElement>) {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
  }

  function handleAddressSubmit(e: React.FormEvent) {
    e.preventDefault();
    setShippingAddress(formData);
  }

  async function handleOrderSubmit() {
    const validationErrors = validate();
    
    if (validationErrors.length > 0) {
      setErrors(validationErrors);
      return;
    }

    setIsSubmitting(true);
    setErrors([]);

    try {
      await apiClient.createOrder({
        customerId: 'current-user-id', // Would come from auth context
        items,
        shippingAddress: shippingAddress!,
        subtotal,
        tax,
        shipping,
        total,
      });

      setOrderConfirmed(true);
      clearOrder();
    } catch (error) {
      setErrors(['Failed to submit order. Please try again.']);
    } finally {
      setIsSubmitting(false);
    }
  }

  if (orderConfirmed) {
    return (
      <div className="p-6 bg-green-50 border border-green-200 rounded">
        <h2 className="text-2xl font-bold text-green-800 mb-2">
          Order Confirmed!
        </h2>
        <p className="text-green-700">
          Thank you for your order. You'll receive a confirmation email shortly.
        </p>
      </div>
    );
  }

  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Checkout</h1>

      {/* Order items summary */}
      <div className="mb-8">
        <h2 className="text-xl font-semibold mb-4">Order Items</h2>
        {items.length === 0 ? (
          <p className="text-gray-600">Your cart is empty</p>
        ) : (
          <ul className="space-y-2">
            {items.map((item) => (
              <li key={item.productId} className="flex justify-between">
                <span>Product {item.productId} × {item.quantity}</span>
                <span>${(item.priceAtPurchase * item.quantity).toFixed(2)}</span>
              </li>
            ))}
          </ul>
        )}
      </div>

      {/* Shipping address form */}
      <form onSubmit={handleAddressSubmit} className="mb-8">
        <h2 className="text-xl font-semibold mb-4">Shipping Address</h2>
        
        <div className="space-y-4">
          <input
            type="text"
            name="street"
            value={formData.street}
            onChange={handleInputChange}
            placeholder="Street Address"
            required
            className="w-full px-4 py-2 border rounded"
          />
          
          <div className="grid grid-cols-2 gap-4">
            <input
              type="text"
              name="city"
              value={formData.city}
              onChange={handleInputChange}
              placeholder="City"
              required
              className="px-4 py-2 border rounded"
            />
            
            <input
              type="text"
              name="state"
              value={formData.state}
              onChange={handleInputChange}
              placeholder="State"
              required
              className="px-4 py-2 border rounded"
            />
          </div>
          
          <div className="grid grid-cols-2 gap-4">
            <input
              type="text"
              name="zipCode"
              value={formData.zipCode}
              onChange={handleInputChange}
              placeholder="ZIP Code"
              required
              className="px-4 py-2 border rounded"
            />
            
            <select
              name="country"
              value={formData.country}
              onChange={handleInputChange}
              className="px-4 py-2 border rounded"
            >
              <option value="US">United States</option>
              <option value="CA">Canada</option>
              <option value="MX">Mexico</option>
            </select>
          </div>
        </div>

        <button
          type="submit"
          className="mt-4 px-6 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
        >
          Calculate Shipping
        </button>
      </form>

      {/* Order summary */}
      {shippingAddress && (
        <div className="mb-8 p-4 bg-gray-50 rounded">
          <h2 className="text-xl font-semibold mb-4">Order Summary</h2>
          
          <div className="space-y-2">
            <div className="flex justify-between">
              <span>Subtotal:</span>
              <span>${subtotal.toFixed(2)}</span>
            </div>
            <div className="flex justify-between">
              <span>Tax:</span>
              <span>${tax.toFixed(2)}</span>
            </div>
            <div className="flex justify-between">
              <span>Shipping:</span>
              <span>{shipping === 0 ? 'FREE' : `$${shipping.toFixed(2)}`}</span>
            </div>
            <div className="flex justify-between font-bold text-lg pt-2 border-t">
              <span>Total:</span>
              <span>${total.toFixed(2)}</span>
            </div>
          </div>
        </div>
      )}

      {/* Errors */}
      {errors.length > 0 && (
        <div className="mb-4 p-4 bg-red-50 border border-red-200 rounded">
          <ul className="list-disc list-inside text-red-700">
            {errors.map((error, index) => (
              <li key={index}>{error}</li>
            ))}
          </ul>
        </div>
      )}

      {/* Submit button */}
      <button
        onClick={handleOrderSubmit}
        disabled={!shippingAddress || items.length === 0 || isSubmitting}
        className="w-full px-6 py-3 bg-green-600 text-white rounded-lg hover:bg-green-700 disabled:opacity-50 disabled:cursor-not-allowed"
      >
        {isSubmitting ? 'Processing...' : 'Place Order'}
      </button>
    </div>
  );
}
```

**Why this matters**:

The React component is just a thin UI layer. It:
- Displays data from the store
- Handles user input
- Delegates all logic to the business layer

If you migrate from React to Svelte, you rewrite this component, but:
- `PricingService` stays the same
- `OrderValidator` stays the same
- `OrderApiClient` stays the same (or needs minimal changes)

**Migration effort**:

```
Total codebase: 1000 lines
- Business logic: 400 lines (0% rewrite needed)
- Adapters: 300 lines (20% rewrite needed)
- UI components: 300 lines (100% rewrite needed)

Total rewrite: ~360 lines (36% of codebase)
```

Compare this to a typical React app where business logic is mixed into components:

```
Typical React app: 1000 lines
- Mixed business logic + UI: 1000 lines (80-100% rewrite needed)

Total rewrite: ~800-1000 lines (80-100% of codebase)
```

By separating concerns, you reduce migration effort by 50-70%.

### The Testing Advantage

Framework-agnostic business logic is also **easier to test**. You don't need React Testing Library, jsdom, or any React-specific tooling.

**Testing business logic** (pure unit tests):

```typescript
// src/domain/services/PricingService.test.ts

import { describe, it, expect } from 'vitest';
import { PricingService } from './PricingService';
import type { OrderItem, ShippingAddress } from '../models/Order';

describe('PricingService', () => {
  const service = new PricingService();

  const sampleItems: OrderItem[] = [
    { productId: '1', quantity: 2, priceAtPurchase: 10 },
    { productId: '2', quantity: 1, priceAtPurchase: 30 },
  ];

  const usAddress: ShippingAddress = {
    street: '123 Main St',
    city: 'New York',
    state: 'NY',
    zipCode: '10001',
    country: 'US',
  };

  describe('calculateSubtotal', () => {
    it('calculates subtotal correctly', () => {
      const subtotal = service.calculateSubtotal(sampleItems);
      expect(subtotal).toBe(50); // (2 * 10) + (1 * 30)
    });

    it('returns 0 for empty cart', () => {
      const subtotal = service.calculateSubtotal([]);
      expect(subtotal).toBe(0);
    });
  });

  describe('calculateTax', () => {
    it('calculates 8% tax', () => {
      const tax = service.calculateTax(100);
      expect(tax).toBe(8);
    });
  });

  describe('calculateShipping', () => {
    it('charges standard shipping for orders under $50', () => {
      const shipping = service.calculateShipping(40, usAddress);
      expect(shipping).toBe(5.99);
    });

    it('provides free shipping for orders over $50', () => {
      const shipping = service.calculateShipping(60, usAddress);
      expect(shipping).toBe(0);
    });

    it('doubles shipping cost for international orders', () => {
      const intlAddress = { ...usAddress, country: 'CA' };
      const shipping = service.calculateShipping(40, intlAddress);
      expect(shipping).toBe(11.98); // 5.99 * 2
    });
  });

  describe('calculateTotal', () => {
    it('calculates complete order total', () => {
      const result = service.calculateTotal(sampleItems, usAddress);
      
      expect(result.subtotal).toBe(50);
      expect(result.tax).toBe(4); // 8% of 50
      expect(result.shipping).toBe(0); // Free shipping over $50
      expect(result.total).toBe(54); // 50 + 4 + 0
    });
  });
});
```

**Notice**:
- No React imports
- No component rendering
- No DOM manipulation
- Just pure function testing
- Runs in milliseconds
- No flakiness

This is **fast, reliable testing** that will work regardless of your UI framework.

### Patterns for Maintainable Architecture

Based on the reference implementation, here are the key patterns for building maintainable applications:

### Pattern 1: Domain-Driven Design (Lite)

Organize code by **domain concepts**, not technical layers.

**Bad** (organized by technology):

```bash
src/
├── components/
│   ├── OrderForm.tsx
│   ├── ProductList.tsx
│   └── UserProfile.tsx
├── hooks/
│   ├── useOrder.ts
│   ├── useProducts.ts
│   └── useUser.ts
├── api/
│   ├── orders.ts
│   ├── products.ts
│   └── users.ts
└── utils/
    ├── validation.ts
    └── formatting.ts
```

**Good** (organized by domain):

```bash
src/
├── orders/
│   ├── domain/
│   │   ├── Order.ts
│   │   ├── OrderService.ts
│   │   └── orderValidation.ts
│   ├── adapters/
│   │   ├── OrderApiClient.ts
│   │   └── orderStore.ts
│   └── ui/
│       ├── OrderForm.tsx
│       └── OrderSummary.tsx
├── products/
│   ├── domain/
│   │   ├── Product.ts
│   │   └── ProductService.ts
│   ├── adapters/
│   │   └── ProductApiClient.ts
│   └── ui/
│       └── ProductList.tsx
└── users/
    ├── domain/
    │   ├── User.ts
    │   └── UserService.ts
    ├── adapters/
    │   └── UserApiClient.ts
    └── ui/
        └── UserProfile.tsx
```

**Why**: When you need to understand or modify order-related functionality, everything is in one place. When you migrate frameworks, you know exactly which files need rewriting (the `ui/` folders).

### Pattern 2: Dependency Injection

Make dependencies explicit and injectable, not hardcoded.

**Bad** (hardcoded dependencies):

```typescript
// Component directly imports and uses API client
import { OrderApiClient } from '@/api/OrderApiClient';

function OrderForm() {
  const apiClient = new OrderApiClient(); // Hardcoded
  
  async function handleSubmit() {
    await apiClient.createOrder(orderData);
  }
  
  // ...
}
```

**Good** (injected dependencies):

```typescript
// Component receives API client as prop or context
interface OrderFormProps {
  apiClient: OrderApiClient;
}

function OrderForm({ apiClient }: OrderFormProps) {
  async function handleSubmit() {
    await apiClient.createOrder(orderData);
  }
  
  // ...
}

// Or use context for app-wide dependencies
const ApiContext = createContext<OrderApiClient | null>(null);

function OrderForm() {
  const apiClient = useContext(ApiContext);
  if (!apiClient) throw new Error('ApiContext not provided');
  
  // ...
}
```

**Why**: Injected dependencies make testing easier (you can inject mocks) and make it possible to swap implementations without changing component code.

### Pattern 3: Interface-Based Contracts

Define interfaces for adapters, not concrete implementations.

**Bad** (concrete class dependency):

```typescript
import { OrderApiClient } from './OrderApiClient';

function useOrders(client: OrderApiClient) {
  // Tightly coupled to specific implementation
}
```

**Good** (interface dependency):

```typescript
// Define interface
interface IOrderRepository {
  createOrder(draft: OrderDraft): Promise<Order>;
  getOrder(id: string): Promise<Order>;
  updateOrder(id: string, updates: Partial<Order>): Promise<Order>;
}

// Implementation 1: REST API
class OrderApiClient implements IOrderRepository {
  async createOrder(draft: OrderDraft): Promise<Order> {
    // REST API implementation
  }
  // ...
}

// Implementation 2: GraphQL
class OrderGraphQLClient implements IOrderRepository {
  async createOrder(draft: OrderDraft): Promise<Order> {
    // GraphQL implementation
  }
  // ...
}

// Implementation 3: Mock for testing
class MockOrderRepository implements IOrderRepository {
  async createOrder(draft: OrderDraft): Promise<Order> {
    // Return mock data
  }
  // ...
}

// Hook depends on interface, not implementation
function useOrders(repository: IOrderRepository) {
  // Can work with any implementation
}
```

**Why**: You can swap implementations (REST → GraphQL, real → mock) without changing any code that uses the interface.

### Pattern 4: Pure Functions for Business Logic

Keep business logic in pure functions that are easy to test and reason about.

**Bad** (business logic in component):

```tsx
function OrderSummary({ items, address }) {
  // Business logic mixed with UI
  const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const tax = subtotal * 0.08;
  const shipping = subtotal >= 50 ? 0 : address.country === 'US' ? 5.99 : 11.98;
  const total = subtotal + tax + shipping;
  
  return (
    <div>
      <p>Subtotal: ${subtotal}</p>
      <p>Tax: ${tax}</p>
      <p>Shipping: ${shipping}</p>
      <p>Total: ${total}</p>
    </div>
  );
}
```

**Good** (business logic in service):

```tsx
// Business logic in pure function
function calculateOrderTotal(items: OrderItem[], address: ShippingAddress) {
  const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const tax = subtotal * 0.08;
  const shipping = subtotal >= 50 ? 0 : address.country === 'US' ? 5.99 : 11.98;
  const total = subtotal + tax + shipping;
  
  return { subtotal, tax, shipping, total };
}

// Component just displays
function OrderSummary({ items, address }) {
  const { subtotal, tax, shipping, total } = calculateOrderTotal(items, address);
  
  return (
    <div>
      <p>Subtotal: ${subtotal}</p>
      <p>Tax: ${tax}</p>
      <p>Shipping: ${shipping}</p>
      <p>Total: ${total}</p>
    </div>
  );
}
```

**Why**: The business logic can be tested independently, reused in other contexts (server-side, CLI tools), and migrated to other frameworks without changes.

### The Migration Path: When It's Time to Move On

Eventually, you will need to migrate from React. Here's how the architecture we've built makes that manageable:

**Migration checklist**:

1. **Audit your codebase**:
   - Identify framework-agnostic code (should be 40-60% of codebase)
   - Identify adapter code (should be 20-30% of codebase)
   - Identify UI code (should be 20-40% of codebase)

2. **Choose your new framework**:
   - Evaluate based on current constraints (performance, team, ecosystem)
   - Prototype a few components to validate the choice

3. **Migrate in phases**:
   - **Phase 1**: Set up new framework alongside React
   - **Phase 2**: Migrate one domain at a time (e.g., orders first)
   - **Phase 3**: Rewrite adapters for new framework patterns
   - **Phase 4**: Rewrite UI components
   - **Phase 5**: Remove React dependencies

4. **Validate at each phase**:
   - Run existing tests (business logic tests should pass unchanged)
   - Add new tests for new UI components
   - Monitor performance and user experience

**Example migration timeline** (React → Svelte):

```
Week 1-2: Setup and prototyping
- Set up Svelte alongside React
- Prototype 2-3 components to validate approach
- Establish new patterns for Svelte

Week 3-4: Migrate orders domain
- Rewrite OrderForm.tsx → OrderForm.svelte
- Rewrite OrderSummary.tsx → OrderSummary.svelte
- Adapt orderStore.ts for Svelte stores
- Business logic (PricingService, OrderValidator) unchanged

Week 5-6: Migrate products domain
- Rewrite ProductList.tsx → ProductList.svelte
- Adapt productStore.ts for Svelte stores
- Business logic unchanged

Week 7-8: Migrate users domain
- Rewrite UserProfile.tsx → UserProfile.svelte
- Adapt userStore.ts for Svelte stores
- Business logic unchanged

Week 9-10: Cleanup and optimization
- Remove React dependencies
- Optimize bundle size
- Performance testing
- Deploy to production
```

**Total migration time**: 10 weeks for a medium-sized application, with minimal risk because business logic remains stable throughout.

### The Long-Term Mindset

Building maintainable applications requires thinking beyond the current framework:

**Ask yourself**:

1. **If I had to migrate this to a different framework in 5 years, how much would I need to rewrite?**
   - Target: < 50% of codebase

2. **Can I test my business logic without any framework-specific tooling?**
   - Target: Yes, with pure unit tests

3. **Is my business logic reusable in other contexts** (CLI tools, server-side, mobile)?
   - Target: Yes, it's framework-agnostic

4. **How coupled is my code to React-specific patterns?**
   - Target: Only UI components are React-specific

5. **Could a new developer understand the business logic without knowing React?**
   - Target: Yes, it's just TypeScript

If you can answer these questions positively, you've built a maintainable application that will outlast React.

### The Final Lesson: Frameworks Are Tools, Not Foundations

React is a tool. A powerful, well-designed tool that solves real problems. But it's not the foundation of your application—your business logic is.

**The hierarchy**:

```
┌─────────────────────────────────────────────────────────┐
│                    Your Business                         │
│              (The actual value you provide)              │
└─────────────────────────────────────────────────────────┘
                          ↑
                          │
┌─────────────────────────────────────────────────────────┐
│                  Business Logic                          │
│         (Domain models, services, validation)            │
│              (Framework-agnostic)                        │
└─────────────────────────────────────────────────────────┘
                          ↑
                          │
┌─────────────────────────────────────────────────────────┐
│                    Adapters                              │
│         (API clients, state management)                  │
│              (Framework-aware)                           │
└─────────────────────────────────────────────────────────┘
                          ↑
                          │
┌─────────────────────────────────────────────────────────┐
│                   UI Framework                           │
│                    (React)                               │
│              (Replaceable tool)                          │
└─────────────────────────────────────────────────────────┘
```

Build your application from the top down, not the bottom up. Start with your business logic, then add adapters, then add UI. This way, when React eventually becomes legacy, your business logic survives.

**The professional developer's creed**:

> "I build applications that solve problems. I use frameworks as tools to build those applications. When a better tool comes along, I can adopt it without rebuilding my application from scratch."

This is the path to building maintainable applications that outlast framework churn. Not by avoiding frameworks, but by using them wisely—as tools, not foundations.

You've learned React. Now use it to build something that will outlive it.
