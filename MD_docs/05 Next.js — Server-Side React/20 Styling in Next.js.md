# Chapter 20: Styling in Next.js

## Tailwind CSS: the pragmatic choice

## The Problem: Styling at Scale

You've built a Next.js application with Server Components, data fetching, and authentication. Your components work perfectly. But they look like they were designed in 1995.

You need to style your application. Not just make it "not ugly"—you need a styling solution that:

- Works seamlessly with Server Components
- Doesn't cause flash of unstyled content (FOUC)
- Scales from prototype to production
- Doesn't require writing CSS class names for every single element
- Integrates with component libraries
- Supports theming and dark mode

Let's establish our reference implementation: a product catalog that we'll style progressively through this chapter.

### Reference Implementation: E-commerce Product Catalog

We'll build a product listing page with:
- Product cards with images, titles, prices
- Category filters
- Search functionality
- Responsive grid layout
- Dark mode support

**Project Structure**:
```
src/
├── app/
│   ├── products/
│   │   ├── page.tsx          ← Product listing (our focus)
│   │   └── [id]/
│   │       └── page.tsx      ← Product detail
│   ├── layout.tsx
│   └── globals.css
├── components/
│   ├── ProductCard.tsx       ← Individual product display
│   ├── ProductGrid.tsx       ← Grid container
│   └── SearchBar.tsx         ← Search input
└── lib/
    └── products.ts           ← Data fetching
```

Let's start with unstyled components to see the problem clearly.

```typescript
// src/lib/products.ts
export interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  image: string;
  description: string;
}

export async function getProducts(): Promise<Product[]> {
  // Simulating API call
  return [
    {
      id: '1',
      name: 'Wireless Headphones',
      price: 99.99,
      category: 'Electronics',
      image: '/products/headphones.jpg',
      description: 'High-quality wireless headphones with noise cancellation'
    },
    {
      id: '2',
      name: 'Smart Watch',
      price: 299.99,
      category: 'Electronics',
      image: '/products/watch.jpg',
      description: 'Feature-rich smartwatch with health tracking'
    },
    {
      id: '3',
      name: 'Laptop Stand',
      price: 49.99,
      category: 'Accessories',
      image: '/products/stand.jpg',
      description: 'Ergonomic aluminum laptop stand'
    }
  ];
}
```

```tsx
// src/components/ProductCard.tsx
import Image from 'next/image';
import Link from 'next/link';
import { Product } from '@/lib/products';

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  return (
    <Link href={`/products/${product.id}`}>
      <div>
        <Image
          src={product.image}
          alt={product.name}
          width={300}
          height={300}
        />
        <h3>{product.name}</h3>
        <p>{product.category}</p>
        <p>${product.price.toFixed(2)}</p>
      </div>
    </Link>
  );
}
```

```tsx
// src/components/ProductGrid.tsx
import { Product } from '@/lib/products';
import { ProductCard } from './ProductCard';

interface ProductGridProps {
  products: Product[];
}

export function ProductGrid({ products }: ProductGridProps) {
  return (
    <div>
      {products.map((product) => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

```tsx
// src/app/products/page.tsx
import { getProducts } from '@/lib/products';
import { ProductGrid } from '@/components/ProductGrid';

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div>
      <h1>Our Products</h1>
      <ProductGrid products={products} />
    </div>
  );
}
```

### The Failure: Unstyled Components

Run the application and navigate to `/products`.

**Browser Behavior**:
- Products display in a vertical list (no grid)
- Images are full-width, breaking layout
- No spacing between elements
- Text is default browser styling (Times New Roman)
- Links are blue and underlined
- No hover states or visual feedback
- Mobile view is identical to desktop (no responsiveness)

**Visual Evidence**:
```
[Wireless Headphones]
[Full-width image]
Wireless Headphones
Electronics
$99.99

[Smart Watch]
[Full-width image]
Smart Watch
Electronics
$299.99
```

Everything is stacked vertically with minimal spacing. It looks like a document from 1995.

### Diagnostic Analysis: Why Unstyled Components Fail

**What the user experiences**:
- Expected: Modern, grid-based product catalog with visual hierarchy
- Actual: Vertical list of unstyled elements with no visual design

**What we need**:
1. Grid layout for product cards
2. Consistent spacing and typography
3. Visual hierarchy (headings, prices, categories)
4. Responsive design (mobile, tablet, desktop)
5. Interactive states (hover, focus)
6. Professional color scheme

**Why inline styles won't solve this**:
- Inline styles don't support media queries (no responsive design)
- No hover/focus states
- Repetitive code for every element
- No design system or consistency
- Hard to maintain and update

**Why traditional CSS files are problematic**:
- Naming conventions become complex at scale (BEM, SMACSS)
- Unused CSS accumulates over time
- Global namespace conflicts
- Hard to know which styles are safe to delete
- Difficult to co-locate styles with components

**What we need**: A styling solution that provides utility classes for rapid development while maintaining type safety and avoiding the pitfalls of traditional CSS.

## Tailwind CSS: Utility-First Styling

Tailwind CSS is a utility-first CSS framework. Instead of writing custom CSS classes, you compose designs using pre-defined utility classes directly in your JSX.

**Philosophy**:
- Utility classes for every CSS property (`text-center`, `flex`, `bg-blue-500`)
- Compose complex designs from simple utilities
- No naming conventions needed
- Responsive modifiers built-in (`md:flex`, `lg:grid-cols-3`)
- Purges unused CSS automatically in production

### Setting Up Tailwind CSS in Next.js

Next.js has first-class Tailwind support. Let's install it.

```bash
# Install Tailwind CSS and its dependencies
npm install -D tailwindcss postcss autoprefixer

# Initialize Tailwind configuration
npx tailwindcss init -p
```

This creates two files:
- `tailwind.config.js` - Tailwind configuration
- `postcss.config.js` - PostCSS configuration (Tailwind runs through PostCSS)

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

**Critical configuration**: The `content` array tells Tailwind which files to scan for class names. This enables automatic purging of unused CSS in production.

Now add Tailwind's directives to your global CSS file:

```css
/* src/app/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

Import this CSS file in your root layout:

```tsx
// src/app/layout.tsx
import './globals.css';

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

### Iteration 1: Styling with Tailwind Utilities

Let's transform our unstyled components using Tailwind's utility classes.

**Before** (Unstyled):
```tsx
<div>
  <Image src={product.image} alt={product.name} width={300} height={300} />
  <h3>{product.name}</h3>
  <p>{product.category}</p>
  <p>${product.price.toFixed(2)}</p>
</div>
```

**After** (Tailwind styled):

```tsx
// src/components/ProductCard.tsx
import Image from 'next/image';
import Link from 'next/link';
import { Product } from '@/lib/products';

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  return (
    <Link 
      href={`/products/${product.id}`}
      className="group block"
    >
      <div className="overflow-hidden rounded-lg border border-gray-200 bg-white shadow-sm transition-shadow hover:shadow-md">
        {/* Image container with aspect ratio */}
        <div className="relative aspect-square overflow-hidden bg-gray-100">
          <Image
            src={product.image}
            alt={product.name}
            fill
            className="object-cover transition-transform group-hover:scale-105"
          />
        </div>
        
        {/* Content */}
        <div className="p-4">
          <p className="text-sm text-gray-500">{product.category}</p>
          <h3 className="mt-1 text-lg font-semibold text-gray-900 group-hover:text-blue-600">
            {product.name}
          </h3>
          <p className="mt-2 text-xl font-bold text-gray-900">
            ${product.price.toFixed(2)}
          </p>
        </div>
      </div>
    </Link>
  );
}
```

**What changed**:
- `group` class on Link enables hover effects on children
- `overflow-hidden rounded-lg` creates rounded corners with hidden overflow
- `border border-gray-200` adds subtle border
- `shadow-sm hover:shadow-md` adds shadow that increases on hover
- `aspect-square` maintains 1:1 aspect ratio for images
- `fill` on Image makes it fill the container
- `object-cover` ensures image covers area without distortion
- `group-hover:scale-105` scales image on card hover
- `p-4` adds padding (1rem = 16px)
- `text-sm`, `text-lg`, `text-xl` control font sizes
- `font-semibold`, `font-bold` control font weights
- `text-gray-500`, `text-gray-900` control text colors
- `mt-1`, `mt-2` add top margin

Now update the grid container:

```tsx
// src/components/ProductGrid.tsx
import { Product } from '@/lib/products';
import { ProductCard } from './ProductCard';

interface ProductGridProps {
  products: Product[];
}

export function ProductGrid({ products }: ProductGridProps) {
  return (
    <div className="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
      {products.map((product) => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

**Responsive grid breakdown**:
- `grid` enables CSS Grid
- `grid-cols-1` - 1 column on mobile (default)
- `sm:grid-cols-2` - 2 columns on small screens (640px+)
- `lg:grid-cols-3` - 3 columns on large screens (1024px+)
- `xl:grid-cols-4` - 4 columns on extra-large screens (1280px+)
- `gap-6` - 1.5rem (24px) gap between grid items

Update the page layout:

```tsx
// src/app/products/page.tsx
import { getProducts } from '@/lib/products';
import { ProductGrid } from '@/components/ProductGrid';

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div className="mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
      <h1 className="mb-8 text-3xl font-bold text-gray-900">
        Our Products
      </h1>
      <ProductGrid products={products} />
    </div>
  );
}
```

**Layout utilities**:
- `mx-auto` centers the container horizontally
- `max-w-7xl` sets maximum width (80rem = 1280px)
- `px-4 sm:px-6 lg:px-8` responsive horizontal padding
- `py-8` vertical padding (2rem = 32px)
- `mb-8` bottom margin on heading

### Verification: Styled Product Catalog

Restart your development server and navigate to `/products`.

**Browser Behavior**:
- Products display in responsive grid (1/2/3/4 columns based on screen width)
- Cards have rounded corners, borders, and shadows
- Images maintain aspect ratio and scale on hover
- Typography has clear hierarchy (category, name, price)
- Hover states provide visual feedback
- Layout adapts smoothly to different screen sizes

**Expected vs. Actual**:
- ✅ Grid layout works across all screen sizes
- ✅ Visual hierarchy is clear
- ✅ Interactive states provide feedback
- ✅ Professional appearance
- ✅ No flash of unstyled content (CSS is bundled with Next.js)

### Understanding Tailwind's Utility Classes

Let's decode the most common patterns:

**Spacing** (margin and padding):
- `m-4` = margin: 1rem (16px)
- `mt-4` = margin-top: 1rem
- `p-4` = padding: 1rem
- `px-4` = padding-left and padding-right: 1rem
- `py-4` = padding-top and padding-bottom: 1rem

**Sizing**:
- `w-full` = width: 100%
- `h-64` = height: 16rem (256px)
- `max-w-7xl` = max-width: 80rem (1280px)

**Typography**:
- `text-sm` = font-size: 0.875rem (14px)
- `text-lg` = font-size: 1.125rem (18px)
- `font-bold` = font-weight: 700
- `text-gray-900` = color: #111827

**Layout**:
- `flex` = display: flex
- `grid` = display: grid
- `grid-cols-3` = grid-template-columns: repeat(3, minmax(0, 1fr))
- `gap-4` = gap: 1rem

**Responsive modifiers**:
- `sm:` = @media (min-width: 640px)
- `md:` = @media (min-width: 768px)
- `lg:` = @media (min-width: 1024px)
- `xl:` = @media (min-width: 1280px)

**State modifiers**:
- `hover:` = :hover pseudo-class
- `focus:` = :focus pseudo-class
- `group-hover:` = hover on parent with `group` class

### Adding Search and Filters

Let's add a search bar to demonstrate form styling with Tailwind.

```tsx
// src/components/SearchBar.tsx
'use client';

import { useState } from 'react';

interface SearchBarProps {
  onSearch: (query: string) => void;
}

export function SearchBar({ onSearch }: SearchBarProps) {
  const [query, setQuery] = useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    onSearch(query);
  };

  return (
    <form onSubmit={handleSubmit} className="mb-8">
      <div className="flex gap-2">
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Search products..."
          className="flex-1 rounded-lg border border-gray-300 px-4 py-2 focus:border-blue-500 focus:outline-none focus:ring-2 focus:ring-blue-500"
        />
        <button
          type="submit"
          className="rounded-lg bg-blue-600 px-6 py-2 font-semibold text-white transition-colors hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2"
        >
          Search
        </button>
      </div>
    </form>
  );
}
```

**Form styling patterns**:
- `flex gap-2` creates horizontal layout with spacing
- `flex-1` makes input grow to fill available space
- `rounded-lg` rounds corners
- `border border-gray-300` adds border
- `px-4 py-2` adds padding inside input
- `focus:border-blue-500` changes border color on focus
- `focus:outline-none` removes default browser outline
- `focus:ring-2 focus:ring-blue-500` adds custom focus ring
- `bg-blue-600` sets background color
- `hover:bg-blue-700` darkens on hover
- `focus:ring-offset-2` adds space between element and focus ring

### Iteration 2: Category Filters

Add category filter buttons:

```tsx
// src/components/CategoryFilter.tsx
'use client';

interface CategoryFilterProps {
  categories: string[];
  selectedCategory: string | null;
  onSelectCategory: (category: string | null) => void;
}

export function CategoryFilter({
  categories,
  selectedCategory,
  onSelectCategory,
}: CategoryFilterProps) {
  return (
    <div className="mb-6 flex flex-wrap gap-2">
      <button
        onClick={() => onSelectCategory(null)}
        className={`rounded-full px-4 py-2 text-sm font-medium transition-colors ${
          selectedCategory === null
            ? 'bg-blue-600 text-white'
            : 'bg-gray-100 text-gray-700 hover:bg-gray-200'
        }`}
      >
        All
      </button>
      {categories.map((category) => (
        <button
          key={category}
          onClick={() => onSelectCategory(category)}
          className={`rounded-full px-4 py-2 text-sm font-medium transition-colors ${
            selectedCategory === category
              ? 'bg-blue-600 text-white'
              : 'bg-gray-100 text-gray-700 hover:bg-gray-200'
          }`}
        >
          {category}
        </button>
      ))}
    </div>
  );
}
```

**Conditional styling pattern**:
- Template literal with ternary operator for dynamic classes
- `rounded-full` creates pill-shaped buttons
- Different styles for selected vs. unselected state
- `flex-wrap` allows buttons to wrap on small screens

### When to Apply Tailwind CSS

**What it optimizes for**:
- Rapid development and prototyping
- Consistency through design system
- Automatic purging of unused CSS
- No naming conventions needed
- Co-location of styles with components

**What it sacrifices**:
- Initial learning curve (memorizing utility classes)
- Verbose className strings
- Harder to read for developers unfamiliar with Tailwind

**When to choose Tailwind**:
- Building new applications from scratch
- Need rapid iteration and prototyping
- Want consistent design system
- Team is comfortable with utility-first approach
- Using component libraries that support Tailwind (shadcn/ui, Headless UI)

**When to avoid Tailwind**:
- Existing codebase with established CSS architecture
- Team strongly prefers traditional CSS
- Need very custom, artistic designs (though Tailwind is flexible)
- Working with designers who provide pixel-perfect mockups in traditional CSS

### Code Characteristics

**Setup complexity**: Low
- Single npm install
- Minimal configuration
- Works out of the box with Next.js

**Maintenance burden**: Low
- No CSS files to maintain
- Styles co-located with components
- Automatic purging prevents bloat

**Performance impact**: Excellent
- Tiny production bundle (only used utilities)
- No runtime JavaScript
- Optimized by PostCSS

### Common Failure Modes and Their Signatures

#### Symptom: Styles not applying

**Browser behavior**:
Classes are in the HTML but have no effect

**Console pattern**:
No errors (Tailwind silently ignores unknown classes)

**DevTools clues**:
- Element has class names in HTML
- No corresponding CSS rules in Styles panel
- Check for typos: `text-centre` vs. `text-center`

**Root cause**: Typo in class name or class not in Tailwind's default configuration

**Solution**: Check Tailwind documentation for correct class name, or extend theme in `tailwind.config.js`

#### Symptom: Styles work in development but not production

**Browser behavior**:
Styles disappear after `npm run build`

**Console pattern**:
No errors

**Root cause**: File not included in Tailwind's `content` configuration

**Solution**: Add file path pattern to `content` array in `tailwind.config.js`

#### Symptom: Custom colors not working

**Browser behavior**:
`bg-brand-500` doesn't apply

**Root cause**: Custom colors must be defined in theme configuration

**Solution**:

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        brand: {
          500: '#3B82F6',
          600: '#2563EB',
        },
      },
    },
  },
}
```

### Debugging Workflow: Tailwind Issues

**Step 1: Verify Tailwind is loaded**
- Open DevTools → Elements
- Inspect any element
- Check Styles panel for Tailwind utility classes
- If no Tailwind classes exist, check `globals.css` import

**Step 2: Check class name spelling**
- Tailwind silently ignores typos
- Common mistakes: `center` vs. `centre`, `gray` vs. `grey`
- Use VS Code Tailwind IntelliSense extension for autocomplete

**Step 3: Verify content configuration**
- Check `tailwind.config.js` content array
- Ensure your file paths are included
- Restart dev server after config changes

**Step 4: Check for class conflicts**
- Later classes override earlier ones
- `className="text-red-500 text-blue-500"` → blue wins
- Use conditional logic carefully

**Step 5: Inspect computed styles**
- DevTools → Elements → Computed tab
- See final CSS values
- Identify which rule is actually applied

## CSS Modules as a fallback

## When Tailwind Isn't Enough

Tailwind is excellent for most styling needs, but sometimes you need:
- Complex animations with keyframes
- Pseudo-elements (::before, ::after) with content
- Very specific CSS that doesn't map to utilities
- Scoped styles without verbose className strings

CSS Modules provide scoped CSS with traditional syntax. They're built into Next.js with zero configuration.

### The Failure: Complex Animations in Tailwind

Let's add a loading skeleton to our product cards. We want a shimmer animation that sweeps across the card.

**Attempt with Tailwind**:

```tsx
// src/components/ProductCardSkeleton.tsx - Tailwind attempt
export function ProductCardSkeleton() {
  return (
    <div className="overflow-hidden rounded-lg border border-gray-200 bg-white shadow-sm">
      <div className="aspect-square animate-pulse bg-gray-200" />
      <div className="p-4">
        <div className="h-4 w-20 animate-pulse rounded bg-gray-200" />
        <div className="mt-2 h-6 w-32 animate-pulse rounded bg-gray-200" />
        <div className="mt-2 h-6 w-24 animate-pulse rounded bg-gray-200" />
      </div>
    </div>
  );
}
```

**Browser Behavior**:
- Elements pulse (fade in/out)
- No shimmer effect
- Looks generic, not polished

**Limitation**: Tailwind's `animate-pulse` is a simple opacity animation. Creating a custom shimmer effect requires:
- Custom keyframes
- Gradient backgrounds
- Pseudo-elements for the shimmer overlay

**What we need**: CSS Modules for complex, custom animations.

### CSS Modules: Scoped Traditional CSS

CSS Modules automatically scope CSS class names to prevent conflicts. Each component gets its own CSS file.

**File naming convention**: `ComponentName.module.css`

Let's create a proper shimmer skeleton:

```css
/* src/components/ProductCardSkeleton.module.css */
.skeleton {
  overflow: hidden;
  border-radius: 0.5rem;
  border: 1px solid #e5e7eb;
  background-color: white;
  box-shadow: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
}

.imageContainer {
  position: relative;
  aspect-ratio: 1;
  background: linear-gradient(90deg, #f3f4f6 25%, #e5e7eb 50%, #f3f4f6 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}

.content {
  padding: 1rem;
}

.categoryBar {
  height: 1rem;
  width: 5rem;
  border-radius: 0.25rem;
  background: linear-gradient(90deg, #f3f4f6 25%, #e5e7eb 50%, #f3f4f6 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}

.titleBar {
  margin-top: 0.5rem;
  height: 1.5rem;
  width: 8rem;
  border-radius: 0.25rem;
  background: linear-gradient(90deg, #f3f4f6 25%, #e5e7eb 50%, #f3f4f6 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  animation-delay: 0.1s;
}

.priceBar {
  margin-top: 0.5rem;
  height: 1.5rem;
  width: 6rem;
  border-radius: 0.25rem;
  background: linear-gradient(90deg, #f3f4f6 25%, #e5e7eb 50%, #f3f4f6 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  animation-delay: 0.2s;
}

@keyframes shimmer {
  0% {
    background-position: 200% 0;
  }
  100% {
    background-position: -200% 0;
  }
}
```

```tsx
// src/components/ProductCardSkeleton.tsx
import styles from './ProductCardSkeleton.module.css';

export function ProductCardSkeleton() {
  return (
    <div className={styles.skeleton}>
      <div className={styles.imageContainer} />
      <div className={styles.content}>
        <div className={styles.categoryBar} />
        <div className={styles.titleBar} />
        <div className={styles.priceBar} />
      </div>
    </div>
  );
}
```

**How CSS Modules work**:
1. Import CSS file as JavaScript object: `import styles from './Component.module.css'`
2. Access classes as properties: `styles.skeleton`, `styles.imageContainer`
3. Next.js automatically generates unique class names: `ProductCardSkeleton_skeleton__a1b2c3`
4. Styles are scoped to this component only

**Verification**:

**Browser Behavior**:
- Shimmer animation sweeps across skeleton elements
- Staggered animation (category → title → price)
- Smooth, professional loading state

**DevTools Evidence**:
```html
<div class="ProductCardSkeleton_skeleton__a1b2c3">
  <div class="ProductCardSkeleton_imageContainer__d4e5f6"></div>
  ...
</div>
```

Class names are automatically scoped with hash suffix.

### Iteration 3: Combining Tailwind and CSS Modules

You can use both approaches in the same component. Use Tailwind for standard utilities and CSS Modules for complex custom styles.

**Pattern**: Combine class names with template literals:

```tsx
// src/components/ProductCard.tsx - Hybrid approach
import Image from 'next/image';
import Link from 'next/link';
import { Product } from '@/lib/products';
import styles from './ProductCard.module.css';

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  return (
    <Link 
      href={`/products/${product.id}`}
      className="group block"
    >
      <div className="overflow-hidden rounded-lg border border-gray-200 bg-white shadow-sm transition-shadow hover:shadow-md">
        <div className={`relative aspect-square overflow-hidden bg-gray-100 ${styles.imageContainer}`}>
          <Image
            src={product.image}
            alt={product.name}
            fill
            className="object-cover transition-transform group-hover:scale-105"
          />
          {product.isNew && (
            <span className={styles.badge}>New</span>
          )}
        </div>
        
        <div className="p-4">
          <p className="text-sm text-gray-500">{product.category}</p>
          <h3 className="mt-1 text-lg font-semibold text-gray-900 group-hover:text-blue-600">
            {product.name}
          </h3>
          <p className="mt-2 text-xl font-bold text-gray-900">
            ${product.price.toFixed(2)}
          </p>
        </div>
      </div>
    </Link>
  );
}
```

```css
/* src/components/ProductCard.module.css */
.imageContainer {
  position: relative;
}

.badge {
  position: absolute;
  top: 0.5rem;
  right: 0.5rem;
  padding: 0.25rem 0.75rem;
  background-color: #3b82f6;
  color: white;
  font-size: 0.75rem;
  font-weight: 600;
  border-radius: 9999px;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.badge::before {
  content: '✨';
  margin-right: 0.25rem;
}
```

**Why this works**:
- Tailwind handles standard utilities (spacing, colors, typography)
- CSS Modules handle complex custom styles (badge with pseudo-element)
- Both class names coexist: `className="tailwind-classes ${styles.cssModule}"`

### When to Use CSS Modules

**What it optimizes for**:
- Complex animations with keyframes
- Pseudo-elements with content
- Scoped styles without verbose class names
- Traditional CSS workflow
- Gradual migration from existing CSS

**What it sacrifices**:
- Separate CSS files to maintain
- Manual scoping (though automatic)
- No design system utilities (unless you build them)

**When to choose CSS Modules**:
- Need complex animations or pseudo-elements
- Team prefers traditional CSS syntax
- Migrating from existing CSS codebase
- Specific styles that don't map to Tailwind utilities
- Want scoped styles without Tailwind's verbosity

**When to avoid CSS Modules**:
- Standard UI components (Tailwind is faster)
- Need design system consistency
- Want minimal CSS maintenance
- Prefer utility-first approach

### Code Characteristics

**Setup complexity**: Zero
- Built into Next.js
- No configuration needed
- Just create `.module.css` files

**Maintenance burden**: Medium
- Separate CSS files to maintain
- Need to manage class name imports
- Can accumulate unused styles

**Performance impact**: Good
- Scoped CSS prevents conflicts
- Automatic code splitting per component
- No runtime JavaScript

### Common Failure Modes and Their Signatures

#### Symptom: Styles not applying

**Browser behavior**:
Component renders but has no styles

**Console pattern**:
No errors

**DevTools clues**:
- Element has no class attribute
- Or class name is undefined: `class="undefined"`

**Root cause**: Forgot to import CSS Module or typo in class name

**Solution**:

```tsx
// ❌ Wrong - forgot import
export function Component() {
  return <div className={styles.container}>...</div>;
}

// ✅ Correct - import CSS Module
import styles from './Component.module.css';

export function Component() {
  return <div className={styles.container}>...</div>;
}
```

#### Symptom: Class name collision

**Browser behavior**:
Styles from different components interfere with each other

**Root cause**: Using regular `.css` file instead of `.module.css`

**Solution**: Rename file to `.module.css` and import as object

#### Symptom: Global styles not working

**Browser behavior**:
Global styles (like body, html) don't apply from CSS Module

**Root cause**: CSS Modules scope all classes, including global selectors

**Solution**: Use `:global()` wrapper or put global styles in `globals.css`

```css
/* Component.module.css */
/* ❌ Wrong - this gets scoped */
body {
  margin: 0;
}

/* ✅ Correct - use :global() */
:global(body) {
  margin: 0;
}

/* Or better - put in globals.css */
```

### Debugging Workflow: CSS Modules Issues

**Step 1: Verify import**
- Check that CSS Module is imported
- Verify file name ends with `.module.css`
- Check import path is correct

**Step 2: Inspect generated class names**
- Open DevTools → Elements
- Check class attribute on element
- Should see scoped name: `Component_className__hash`

**Step 3: Check Styles panel**
- DevTools → Elements → Styles
- Verify CSS rules are present
- Check if rules are being overridden

**Step 4: Verify class name exists in CSS**
- Check for typos: `styles.container` vs. `styles.contianer`
- Use TypeScript for autocomplete (CSS Modules are typed)

**Step 5: Check specificity conflicts**
- CSS Modules have same specificity as regular classes
- Global styles or Tailwind might override
- Use DevTools to see which rule wins

## shadcn/ui: pre-built components done right

## The Problem: Building UI Components from Scratch

You've styled your product catalog with Tailwind. Now you need:
- A modal dialog for product details
- A dropdown menu for user actions
- A toast notification system
- Form inputs with validation states
- Accessible, keyboard-navigable components

You could build these from scratch, but:
- Accessibility is hard (ARIA attributes, keyboard navigation, focus management)
- Edge cases are numerous (click outside, escape key, focus trapping)
- Animations and transitions require careful coordination
- Testing across browsers and devices is time-consuming

**The traditional solution**: Component libraries like Material-UI, Ant Design, Chakra UI.

**The problem with traditional libraries**:
- Heavy bundle size (entire library even if you use 3 components)
- Opinionated styling that's hard to customize
- Runtime JavaScript for theming
- Vendor lock-in (hard to migrate away)
- Often conflict with Tailwind

### shadcn/ui: A Different Approach

shadcn/ui is not a component library you install. It's a collection of **copy-paste components** that you own.

**Philosophy**:
- Copy component code into your project
- Components are yours to modify
- Built with Radix UI (accessible primitives)
- Styled with Tailwind CSS
- No runtime dependencies (except Radix)
- No vendor lock-in

**How it works**:
1. Run CLI command to add component
2. Component code is copied to your `components/ui` directory
3. You own the code and can modify it
4. Import and use like any other component

### Setting Up shadcn/ui

First, initialize shadcn/ui in your Next.js project:

```bash
# Initialize shadcn/ui
npx shadcn-ui@latest init
```

This will prompt you with configuration questions:

```
Would you like to use TypeScript? › Yes
Which style would you like to use? › Default
Which color would you like to use as base color? › Slate
Where is your global CSS file? › src/app/globals.css
Would you like to use CSS variables for colors? › Yes
Where is your tailwind.config.js located? › tailwind.config.js
Configure the import alias for components? › @/components
Configure the import alias for utils? › @/lib/utils
```

This creates:
- `components/ui/` directory for shadcn components
- `lib/utils.ts` with utility functions
- Updates `tailwind.config.js` with shadcn theme
- Updates `globals.css` with CSS variables

### Iteration 4: Adding a Dialog Component

Let's add a modal dialog to show product details. First, install the dialog component:

```bash
# Add dialog component
npx shadcn-ui@latest add dialog
```

This copies the dialog component code to `components/ui/dialog.tsx`. Let's examine what was added:

```tsx
// src/components/ui/dialog.tsx (generated by shadcn)
import * as React from "react"
import * as DialogPrimitive from "@radix-ui/react-dialog"
import { X } from "lucide-react"
import { cn } from "@/lib/utils"

const Dialog = DialogPrimitive.Root
const DialogTrigger = DialogPrimitive.Trigger
const DialogPortal = DialogPrimitive.Portal
const DialogClose = DialogPrimitive.Close

const DialogOverlay = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Overlay>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Overlay>
>(({ className, ...props }, ref) => (
  <DialogPrimitive.Overlay
    ref={ref}
    className={cn(
      "fixed inset-0 z-50 bg-black/80 data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0",
      className
    )}
    {...props}
  />
))
DialogOverlay.displayName = DialogPrimitive.Overlay.displayName

const DialogContent = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Content>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Content>
>(({ className, children, ...props }, ref) => (
  <DialogPortal>
    <DialogOverlay />
    <DialogPrimitive.Content
      ref={ref}
      className={cn(
        "fixed left-[50%] top-[50%] z-50 grid w-full max-w-lg translate-x-[-50%] translate-y-[-50%] gap-4 border bg-background p-6 shadow-lg duration-200 data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 data-[state=closed]:slide-out-to-left-1/2 data-[state=closed]:slide-out-to-top-[48%] data-[state=open]:slide-in-from-left-1/2 data-[state=open]:slide-in-from-top-[48%] sm:rounded-lg",
        className
      )}
      {...props}
    >
      {children}
      <DialogPrimitive.Close className="absolute right-4 top-4 rounded-sm opacity-70 ring-offset-background transition-opacity hover:opacity-100 focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2 disabled:pointer-events-none data-[state=open]:bg-accent data-[state=open]:text-muted-foreground">
        <X className="h-4 w-4" />
        <span className="sr-only">Close</span>
      </DialogPrimitive.Close>
    </DialogPrimitive.Content>
  </DialogPortal>
))
DialogContent.displayName = DialogPrimitive.Content.displayName

// ... more component exports

export {
  Dialog,
  DialogPortal,
  DialogOverlay,
  DialogClose,
  DialogTrigger,
  DialogContent,
  DialogHeader,
  DialogFooter,
  DialogTitle,
  DialogDescription,
}
```

**What this gives you**:
- Accessible dialog built on Radix UI primitives
- Keyboard navigation (Escape to close, Tab to cycle focus)
- Focus trapping (focus stays inside dialog)
- Scroll locking (body doesn't scroll when dialog is open)
- Smooth animations (fade in/out, zoom, slide)
- Styled with Tailwind utilities
- Fully customizable (it's your code now)

Now create a product detail dialog:

```tsx
// src/components/ProductDetailDialog.tsx
'use client';

import Image from 'next/image';
import { Product } from '@/lib/products';
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog';

interface ProductDetailDialogProps {
  product: Product;
  children: React.ReactNode;
}

export function ProductDetailDialog({ product, children }: ProductDetailDialogProps) {
  return (
    <Dialog>
      <DialogTrigger asChild>
        {children}
      </DialogTrigger>
      <DialogContent className="max-w-3xl">
        <DialogHeader>
          <DialogTitle>{product.name}</DialogTitle>
          <DialogDescription>{product.category}</DialogDescription>
        </DialogHeader>
        
        <div className="grid gap-6 md:grid-cols-2">
          <div className="relative aspect-square overflow-hidden rounded-lg bg-gray-100">
            <Image
              src={product.image}
              alt={product.name}
              fill
              className="object-cover"
            />
          </div>
          
          <div className="flex flex-col gap-4">
            <div>
              <p className="text-3xl font-bold text-gray-900">
                ${product.price.toFixed(2)}
              </p>
            </div>
            
            <div>
              <h3 className="mb-2 text-sm font-semibold text-gray-900">
                Description
              </h3>
              <p className="text-sm text-gray-600">
                {product.description}
              </p>
            </div>
            
            <button className="mt-auto rounded-lg bg-blue-600 px-6 py-3 font-semibold text-white transition-colors hover:bg-blue-700">
              Add to Cart
            </button>
          </div>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

Update the ProductCard to use the dialog:

```tsx
// src/components/ProductCard.tsx
import Image from 'next/image';
import { Product } from '@/lib/products';
import { ProductDetailDialog } from './ProductDetailDialog';

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  return (
    <ProductDetailDialog product={product}>
      <div className="group cursor-pointer">
        <div className="overflow-hidden rounded-lg border border-gray-200 bg-white shadow-sm transition-shadow hover:shadow-md">
          <div className="relative aspect-square overflow-hidden bg-gray-100">
            <Image
              src={product.image}
              alt={product.name}
              fill
              className="object-cover transition-transform group-hover:scale-105"
            />
          </div>
          
          <div className="p-4">
            <p className="text-sm text-gray-500">{product.category}</p>
            <h3 className="mt-1 text-lg font-semibold text-gray-900 group-hover:text-blue-600">
              {product.name}
            </h3>
            <p className="mt-2 text-xl font-bold text-gray-900">
              ${product.price.toFixed(2)}
            </p>
          </div>
        </div>
      </div>
    </ProductDetailDialog>
  );
}
```

**Verification**:

**Browser Behavior**:
- Click product card → dialog opens with smooth animation
- Background darkens (overlay)
- Body scroll is locked
- Press Escape → dialog closes
- Click outside dialog → dialog closes
- Click X button → dialog closes
- Tab key cycles through focusable elements inside dialog
- Focus returns to trigger element when closed

**Accessibility Evidence**:
```html
<div role="dialog" aria-modal="true" aria-labelledby="dialog-title" aria-describedby="dialog-description">
  <h2 id="dialog-title">Wireless Headphones</h2>
  <p id="dialog-description">Electronics</p>
  ...
</div>
```

Proper ARIA attributes for screen readers.

### Adding More Components

Let's add a toast notification system for "Add to Cart" feedback:

```bash
# Add toast component
npx shadcn-ui@latest add toast
```

This adds:
- `components/ui/toast.tsx` - Toast component
- `components/ui/toaster.tsx` - Toast container
- `components/ui/use-toast.ts` - Hook for showing toasts

Add the Toaster to your root layout:

```tsx
// src/app/layout.tsx
import './globals.css';
import { Toaster } from '@/components/ui/toaster';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        {children}
        <Toaster />
      </body>
    </html>
  );
}
```

Now use the toast in the product detail dialog:

```tsx
// src/components/ProductDetailDialog.tsx
'use client';

import Image from 'next/image';
import { Product } from '@/lib/products';
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog';
import { useToast } from '@/components/ui/use-toast';

interface ProductDetailDialogProps {
  product: Product;
  children: React.ReactNode;
}

export function ProductDetailDialog({ product, children }: ProductDetailDialogProps) {
  const { toast } = useToast();

  const handleAddToCart = () => {
    toast({
      title: "Added to cart",
      description: `${product.name} has been added to your cart.`,
    });
  };

  return (
    <Dialog>
      <DialogTrigger asChild>
        {children}
      </DialogTrigger>
      <DialogContent className="max-w-3xl">
        <DialogHeader>
          <DialogTitle>{product.name}</DialogTitle>
          <DialogDescription>{product.category}</DialogDescription>
        </DialogHeader>
        
        <div className="grid gap-6 md:grid-cols-2">
          <div className="relative aspect-square overflow-hidden rounded-lg bg-gray-100">
            <Image
              src={product.image}
              alt={product.name}
              fill
              className="object-cover"
            />
          </div>
          
          <div className="flex flex-col gap-4">
            <div>
              <p className="text-3xl font-bold text-gray-900">
                ${product.price.toFixed(2)}
              </p>
            </div>
            
            <div>
              <h3 className="mb-2 text-sm font-semibold text-gray-900">
                Description
              </h3>
              <p className="text-sm text-gray-600">
                {product.description}
              </p>
            </div>
            
            <button 
              onClick={handleAddToCart}
              className="mt-auto rounded-lg bg-blue-600 px-6 py-3 font-semibold text-white transition-colors hover:bg-blue-700"
            >
              Add to Cart
            </button>
          </div>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

**Verification**:

**Browser Behavior**:
- Click "Add to Cart" → toast notification appears in bottom-right corner
- Toast shows product name
- Toast auto-dismisses after 5 seconds
- Multiple toasts stack vertically
- Smooth slide-in animation

### Iteration 5: Form Components

Add form components for a newsletter signup:

```bash
# Add form components
npx shadcn-ui@latest add button input label
```

```tsx
// src/components/NewsletterSignup.tsx
'use client';

import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { useToast } from '@/components/ui/use-toast';

export function NewsletterSignup() {
  const [email, setEmail] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const { toast } = useToast();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setIsLoading(true);

    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 1000));

    toast({
      title: "Subscribed!",
      description: "You've been added to our newsletter.",
    });

    setEmail('');
    setIsLoading(false);
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div className="space-y-2">
        <Label htmlFor="email">Email address</Label>
        <Input
          id="email"
          type="email"
          placeholder="you@example.com"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
      </div>
      <Button type="submit" disabled={isLoading} className="w-full">
        {isLoading ? 'Subscribing...' : 'Subscribe'}
      </Button>
    </form>
  );
}
```

### When to Apply shadcn/ui

**What it optimizes for**:
- Accessible, production-ready components
- Full customization (you own the code)
- No vendor lock-in
- Tailwind-first styling
- Type-safe components
- Small bundle size (only what you use)

**What it sacrifices**:
- Initial setup time (CLI commands for each component)
- More code in your repository
- Manual updates (no npm update)
- Need to understand component internals for deep customization

**When to choose shadcn/ui**:
- Building production applications
- Need accessible components
- Want full control over component code
- Using Tailwind CSS
- Prefer copy-paste over npm install
- Want to learn from well-written component code

**When to avoid shadcn/ui**:
- Prototyping (too much setup)
- Need frequent component updates from maintainers
- Team unfamiliar with Radix UI primitives
- Prefer traditional component libraries

### Code Characteristics

**Setup complexity**: Medium
- CLI initialization required
- Each component needs separate install
- Need to understand Radix UI primitives

**Maintenance burden**: Medium
- Components live in your codebase
- Manual updates when shadcn releases improvements
- Need to maintain component code yourself

**Performance impact**: Excellent
- Only bundle components you use
- No runtime theming overhead
- Tree-shakeable
- Radix UI is lightweight

### Common Failure Modes and Their Signatures

#### Symptom: Component not found

**Browser behavior**:
Import error in console

**Console pattern**:
```
Module not found: Can't resolve '@/components/ui/dialog'
```

**Root cause**: Forgot to install component with CLI

**Solution**: Run `npx shadcn-ui@latest add dialog`

#### Symptom: Styles not applying to shadcn components

**Browser behavior**:
Components render but look unstyled

**Root cause**: CSS variables not configured in `globals.css`

**Solution**: Ensure shadcn initialization added CSS variables:

```css
/* src/app/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    /* ... more variables */
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    /* ... more variables */
  }
}
```

#### Symptom: Dialog doesn't close on Escape

**Browser behavior**:
Dialog stays open when pressing Escape key

**Root cause**: Event handler preventing default behavior

**Solution**: Don't call `e.preventDefault()` on keyboard events in dialog content

#### Symptom: Toast doesn't appear

**Browser behavior**:
`toast()` called but nothing shows

**Root cause**: Forgot to add `<Toaster />` to layout

**Solution**: Add Toaster component to root layout

### Debugging Workflow: shadcn/ui Issues

**Step 1: Verify component installation**
- Check `components/ui/` directory for component file
- If missing, run `npx shadcn-ui@latest add [component]`

**Step 2: Check CSS variables**
- Open DevTools → Elements → Computed
- Verify CSS variables are defined (e.g., `--background`)
- If missing, check `globals.css` has shadcn variables

**Step 3: Inspect Radix primitives**
- shadcn components wrap Radix UI primitives
- Check Radix UI documentation for behavior
- Verify props are passed correctly

**Step 4: Check for conflicts**
- Global styles might override component styles
- Check specificity in DevTools → Styles panel
- Ensure Tailwind classes aren't conflicting

**Step 5: Review component source**
- Component code is in your project
- Read the implementation in `components/ui/`
- Modify if needed (you own the code)

## Dark mode and theming

## The Problem: Supporting Dark Mode

Your product catalog looks great in light mode. But modern applications need dark mode:
- User preference (many users prefer dark interfaces)
- Accessibility (reduces eye strain in low-light environments)
- Battery savings (on OLED screens)
- Professional appearance

### The Failure: Naive Dark Mode Implementation

Let's try the obvious approach: add a dark mode toggle that changes a class on the body.

```tsx
// src/components/DarkModeToggle.tsx - Naive attempt
'use client';

import { useState } from 'react';

export function DarkModeToggle() {
  const [isDark, setIsDark] = useState(false);

  const toggleDarkMode = () => {
    setIsDark(!isDark);
    document.body.classList.toggle('dark');
  };

  return (
    <button onClick={toggleDarkMode}>
      {isDark ? '☀️' : '🌙'}
    </button>
  );
}
```

```tsx
// src/app/products/page.tsx - Using the toggle
import { getProducts } from '@/lib/products';
import { ProductGrid } from '@/components/ProductGrid';
import { DarkModeToggle } from '@/components/DarkModeToggle';

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div className="mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
      <div className="mb-8 flex items-center justify-between">
        <h1 className="text-3xl font-bold text-gray-900">Our Products</h1>
        <DarkModeToggle />
      </div>
      <ProductGrid products={products} />
    </div>
  );
}
```

**Browser Behavior**:
- Click toggle → `dark` class added to body
- But nothing changes visually
- Text is still black on white
- No dark mode styles applied

### Diagnostic Analysis: Why Naive Dark Mode Fails

**What the user experiences**:
- Expected: Dark background, light text when dark mode enabled
- Actual: No visual change, toggle does nothing

**Browser DevTools Evidence**:
```html
<body class="dark">
  <div class="text-gray-900">Our Products</div>
</body>
```

The `dark` class is present, but no styles respond to it.

**What's missing**:
1. Dark mode variants in Tailwind classes
2. CSS variables that change based on dark mode
3. Persistence (preference resets on page reload)
4. System preference detection
5. Hydration mismatch prevention (server renders light, client wants dark)

**Root cause**: Tailwind's dark mode variants aren't enabled, and we're not using them in our components.

### Configuring Tailwind for Dark Mode

Tailwind supports dark mode through the `dark:` variant. First, enable it in your config:

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  darkMode: 'class', // Enable class-based dark mode
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

**Dark mode strategies**:
- `'class'` - Dark mode enabled when `dark` class is on `html` or `body`
- `'media'` - Dark mode follows system preference (`prefers-color-scheme: dark`)

We use `'class'` for manual control with fallback to system preference.

### Iteration 6: Adding Dark Mode Variants

Update components to include dark mode styles:

```tsx
// src/components/ProductCard.tsx - With dark mode
import Image from 'next/image';
import { Product } from '@/lib/products';
import { ProductDetailDialog } from './ProductDetailDialog';

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  return (
    <ProductDetailDialog product={product}>
      <div className="group cursor-pointer">
        <div className="overflow-hidden rounded-lg border border-gray-200 bg-white shadow-sm transition-shadow hover:shadow-md dark:border-gray-700 dark:bg-gray-800">
          <div className="relative aspect-square overflow-hidden bg-gray-100 dark:bg-gray-700">
            <Image
              src={product.image}
              alt={product.name}
              fill
              className="object-cover transition-transform group-hover:scale-105"
            />
          </div>
          
          <div className="p-4">
            <p className="text-sm text-gray-500 dark:text-gray-400">
              {product.category}
            </p>
            <h3 className="mt-1 text-lg font-semibold text-gray-900 group-hover:text-blue-600 dark:text-gray-100 dark:group-hover:text-blue-400">
              {product.name}
            </h3>
            <p className="mt-2 text-xl font-bold text-gray-900 dark:text-gray-100">
              ${product.price.toFixed(2)}
            </p>
          </div>
        </div>
      </div>
    </ProductDetailDialog>
  );
}
```

**Dark mode pattern**:
- Add `dark:` prefix to any utility class
- `bg-white dark:bg-gray-800` - white in light mode, dark gray in dark mode
- `text-gray-900 dark:text-gray-100` - dark text in light mode, light text in dark mode
- `border-gray-200 dark:border-gray-700` - lighter border in light mode, darker in dark mode

Update the page layout:

```tsx
// src/app/products/page.tsx - With dark mode
import { getProducts } from '@/lib/products';
import { ProductGrid } from '@/components/ProductGrid';
import { DarkModeToggle } from '@/components/DarkModeToggle';

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div className="min-h-screen bg-white dark:bg-gray-900">
      <div className="mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
        <div className="mb-8 flex items-center justify-between">
          <h1 className="text-3xl font-bold text-gray-900 dark:text-gray-100">
            Our Products
          </h1>
          <DarkModeToggle />
        </div>
        <ProductGrid products={products} />
      </div>
    </div>
  );
}
```

**Verification**:

**Browser Behavior**:
- Click toggle → background turns dark, text turns light
- Product cards have dark backgrounds
- Borders are visible but subtle
- Hover states still work
- Images remain unchanged (correct)

**But there's still a problem**: Refresh the page and dark mode resets. The preference isn't persisted.

### The Failure: Hydration Mismatch

Let's add localStorage persistence:

```tsx
// src/components/DarkModeToggle.tsx - With localStorage (BROKEN)
'use client';

import { useState, useEffect } from 'react';

export function DarkModeToggle() {
  const [isDark, setIsDark] = useState(false);

  useEffect(() => {
    // Load preference from localStorage
    const stored = localStorage.getItem('darkMode');
    const isDarkMode = stored === 'true';
    setIsDark(isDarkMode);
    document.documentElement.classList.toggle('dark', isDarkMode);
  }, []);

  const toggleDarkMode = () => {
    const newValue = !isDark;
    setIsDark(newValue);
    localStorage.setItem('darkMode', String(newValue));
    document.documentElement.classList.toggle('dark', newValue);
  };

  return (
    <button 
      onClick={toggleDarkMode}
      className="rounded-lg p-2 hover:bg-gray-100 dark:hover:bg-gray-800"
    >
      {isDark ? '☀️' : '🌙'}
    </button>
  );
}
```

**Browser Behavior**:
- Enable dark mode → works
- Refresh page → **flash of light mode before dark mode applies**
- Console shows warning

**Browser Console**:
```
Warning: Prop `className` did not match. Server: "bg-white" Client: "bg-white dark:bg-gray-900"
```

### Diagnostic Analysis: Hydration Mismatch

**What the user experiences**:
- Expected: Dark mode persists across page loads without flash
- Actual: Brief flash of light mode, then dark mode applies

**What's happening**:
1. Server renders HTML with light mode (no access to localStorage)
2. HTML sent to browser shows light mode
3. React hydrates on client
4. useEffect runs, reads localStorage, applies dark mode
5. React sees mismatch between server HTML and client state
6. User sees flash as DOM updates

**React DevTools Evidence**:
- Component tree shows hydration warning
- Server-rendered HTML doesn't match client-rendered HTML

**Root cause**: Server can't access localStorage (it's browser-only). Server always renders light mode, client might need dark mode.

**What we need**: A way to apply dark mode before React hydrates, preventing the flash.

### The Solution: next-themes

The `next-themes` library solves hydration mismatches by:
1. Injecting a script before React hydrates
2. Reading preference from localStorage
3. Applying dark mode class immediately
4. Preventing flash of incorrect theme

Install next-themes:

```bash
npm install next-themes
```

Add the ThemeProvider to your root layout:

```tsx
// src/app/layout.tsx
import './globals.css';
import { ThemeProvider } from 'next-themes';
import { Toaster } from '@/components/ui/toaster';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
          {children}
          <Toaster />
        </ThemeProvider>
      </body>
    </html>
  );
}
```

**ThemeProvider props**:
- `attribute="class"` - Adds `dark` class to `html` element
- `defaultTheme="system"` - Follows system preference by default
- `enableSystem` - Allows system preference detection
- `suppressHydrationWarning` on `html` - Prevents React warning (next-themes handles it)

Now create a proper theme toggle:

```tsx
// src/components/ThemeToggle.tsx
'use client';

import { useTheme } from 'next-themes';
import { useEffect, useState } from 'react';
import { Button } from '@/components/ui/button';

export function ThemeToggle() {
  const [mounted, setMounted] = useState(false);
  const { theme, setTheme } = useTheme();

  // useEffect only runs on the client, so we can safely show the UI
  useEffect(() => {
    setMounted(true);
  }, []);

  if (!mounted) {
    // Render placeholder to avoid hydration mismatch
    return (
      <Button variant="ghost" size="icon" disabled>
        <span className="h-5 w-5" />
      </Button>
    );
  }

  return (
    <Button
      variant="ghost"
      size="icon"
      onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}
    >
      {theme === 'dark' ? (
        <span className="text-xl">☀️</span>
      ) : (
        <span className="text-xl">🌙</span>
      )}
      <span className="sr-only">Toggle theme</span>
    </Button>
  );
}
```

**Key pattern**: The `mounted` check prevents hydration mismatch by:
1. Rendering a placeholder on server (disabled button)
2. Rendering actual toggle after client hydration
3. Avoiding mismatch between server and client HTML

Update the products page:

```tsx
// src/app/products/page.tsx
import { getProducts } from '@/lib/products';
import { ProductGrid } from '@/components/ProductGrid';
import { ThemeToggle } from '@/components/ThemeToggle';

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div className="min-h-screen bg-white dark:bg-gray-900">
      <div className="mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
        <div className="mb-8 flex items-center justify-between">
          <h1 className="text-3xl font-bold text-gray-900 dark:text-gray-100">
            Our Products
          </h1>
          <ThemeToggle />
        </div>
        <ProductGrid products={products} />
      </div>
    </div>
  );
}
```

**Verification**:

**Browser Behavior**:
- Click toggle → theme changes instantly
- Refresh page → **no flash**, correct theme applied immediately
- Close browser, reopen → preference persisted
- No hydration warnings in console

**How it works**:
1. next-themes injects blocking script in `<head>`
2. Script runs before React hydrates
3. Reads localStorage, applies theme class
4. React hydrates with correct theme already applied
5. No mismatch, no flash

### Iteration 7: System Preference Detection

Add a three-way toggle: light, dark, system.

```tsx
// src/components/ThemeToggle.tsx - Three-way toggle
'use client';

import { useTheme } from 'next-themes';
import { useEffect, useState } from 'react';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';
import { Button } from '@/components/ui/button';

export function ThemeToggle() {
  const [mounted, setMounted] = useState(false);
  const { theme, setTheme } = useTheme();

  useEffect(() => {
    setMounted(true);
  }, []);

  if (!mounted) {
    return (
      <Button variant="ghost" size="icon" disabled>
        <span className="h-5 w-5" />
      </Button>
    );
  }

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="icon">
          {theme === 'dark' ? (
            <span className="text-xl">🌙</span>
          ) : theme === 'light' ? (
            <span className="text-xl">☀️</span>
          ) : (
            <span className="text-xl">💻</span>
          )}
          <span className="sr-only">Toggle theme</span>
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem onClick={() => setTheme('light')}>
          ☀️ Light
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme('dark')}>
          🌙 Dark
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme('system')}>
          💻 System
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

**Verification**:

**Browser Behavior**:
- Click toggle → dropdown menu appears
- Select "Light" → light mode
- Select "Dark" → dark mode
- Select "System" → follows OS preference
- Change OS theme → app theme updates automatically

### Using CSS Variables for Theming

For more complex theming, use CSS variables that change based on dark mode. This is how shadcn/ui handles theming.

Update your globals.css:

```css
/* src/app/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96.1%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 221.2 83.2% 53.3%;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --primary: 217.2 91.2% 59.8%;
    --primary-foreground: 222.2 47.4% 11.2%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 224.3 76.3% 48%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

**How CSS variables work**:
- Define colors as HSL values (hue, saturation, lightness)
- Light mode: `--background: 0 0% 100%` (white)
- Dark mode: `--background: 222.2 84% 4.9%` (dark blue-gray)
- Use in Tailwind: `bg-background`, `text-foreground`
- Variables automatically switch when `dark` class is applied

Update Tailwind config to use these variables:

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  darkMode: 'class',
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
        secondary: {
          DEFAULT: 'hsl(var(--secondary))',
          foreground: 'hsl(var(--secondary-foreground))',
        },
        muted: {
          DEFAULT: 'hsl(var(--muted))',
          foreground: 'hsl(var(--muted-foreground))',
        },
        accent: {
          DEFAULT: 'hsl(var(--accent))',
          foreground: 'hsl(var(--accent-foreground))',
        },
        card: {
          DEFAULT: 'hsl(var(--card))',
          foreground: 'hsl(var(--card-foreground))',
        },
      },
    },
  },
  plugins: [],
}
```

Now you can use semantic color names:

```tsx
// src/components/ProductCard.tsx - Using semantic colors
import Image from 'next/image';
import { Product } from '@/lib/products';
import { ProductDetailDialog } from './ProductDetailDialog';

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  return (
    <ProductDetailDialog product={product}>
      <div className="group cursor-pointer">
        <div className="overflow-hidden rounded-lg border bg-card shadow-sm transition-shadow hover:shadow-md">
          <div className="relative aspect-square overflow-hidden bg-muted">
            <Image
              src={product.image}
              alt={product.name}
              fill
              className="object-cover transition-transform group-hover:scale-105"
            />
          </div>
          
          <div className="p-4">
            <p className="text-sm text-muted-foreground">
              {product.category}
            </p>
            <h3 className="mt-1 text-lg font-semibold text-card-foreground group-hover:text-primary">
              {product.name}
            </h3>
            <p className="mt-2 text-xl font-bold text-card-foreground">
              ${product.price.toFixed(2)}
            </p>
          </div>
        </div>
      </div>
    </ProductDetailDialog>
  );
}
```

**Benefits of semantic colors**:
- `bg-card` instead of `bg-white dark:bg-gray-800`
- `text-card-foreground` instead of `text-gray-900 dark:text-gray-100`
- Single class name, works in both themes
- Easy to customize entire theme by changing CSS variables
- Consistent color palette across application

### When to Apply Dark Mode

**What it optimizes for**:
- User preference and accessibility
- Modern, professional appearance
- Reduced eye strain
- Battery savings on OLED screens

**What it sacrifices**:
- Additional development time
- More complex styling (need to consider both themes)
- Testing burden (test in both modes)

**When to implement dark mode**:
- Building consumer-facing applications
- Users spend extended time in your app
- Modern, professional brand
- Accessibility is a priority
- Using component library with dark mode support (shadcn/ui)

**When to skip dark mode**:
- Internal tools with limited usage
- Tight deadlines (add later)
- Brand requires specific light theme
- Limited development resources

### Code Characteristics

**Setup complexity**: Low (with next-themes)
- Single npm install
- Wrap app in ThemeProvider
- Add dark mode variants to components

**Maintenance burden**: Medium
- Need to test both themes
- Every new component needs dark mode styles
- CSS variables simplify but require initial setup

**Performance impact**: Minimal
- next-themes adds ~2KB gzipped
- No runtime performance cost
- CSS variables are native browser feature

### Common Failure Modes and Their Signatures

#### Symptom: Flash of incorrect theme

**Browser behavior**:
Brief flash of light mode before dark mode applies

**Root cause**: Not using next-themes or similar solution

**Solution**: Use next-themes with blocking script

#### Symptom: Hydration mismatch warning

**Browser behavior**:
Console warning about className mismatch

**Console pattern**:
```
Warning: Prop `className` did not match. Server: "..." Client: "..."
```

**Root cause**: Rendering theme-dependent content without mounted check

**Solution**: Use mounted state pattern:

```tsx
const [mounted, setMounted] = useState(false);

useEffect(() => {
  setMounted(true);
}, []);

if (!mounted) {
  return <Placeholder />;
}

return <ActualContent />;
```

#### Symptom: Theme doesn't persist

**Browser behavior**:
Theme resets to default on page reload

**Root cause**: Not using next-themes or localStorage

**Solution**: Use next-themes (handles persistence automatically)

#### Symptom: Some components don't respect dark mode

**Browser behavior**:
Most UI is dark, but some components stay light

**Root cause**: Forgot to add dark mode variants to those components

**Solution**: Add `dark:` variants to all color-related classes

### Debugging Workflow: Dark Mode Issues

**Step 1: Verify dark class is applied**
- Open DevTools → Elements
- Check `<html>` element for `class="dark"`
- If missing, check ThemeProvider configuration

**Step 2: Check CSS variables**
- DevTools → Elements → Computed
- Verify CSS variables change when toggling theme
- Light mode: `--background` should be light
- Dark mode: `--background` should be dark

**Step 3: Inspect component styles**
- DevTools → Elements → Styles
- Check if dark mode variants are present
- Look for `.dark .component-class` rules

**Step 4: Test system preference**
- Change OS theme (System Preferences → Appearance)
- Verify app follows system preference when theme is "system"
- Check `window.matchMedia('(prefers-color-scheme: dark)').matches`

**Step 5: Check localStorage**
- DevTools → Application → Local Storage
- Look for `theme` key
- Value should be "light", "dark", or "system"

## The Styling Journey: From Unstyled to Production

## The Complete Journey

Let's trace the evolution of our product catalog from unstyled components to a production-ready, themed application.

### The Styling Evolution Table

| Iteration | Approach | Result | Bundle Impact | Maintenance |
|-----------|----------|--------|---------------|-------------|
| 0 | No styling | Unstyled, 1995 aesthetic | 0 KB | None |
| 1 | Tailwind utilities | Modern, responsive grid | ~8 KB (purged) | Low |
| 2 | CSS Modules for animations | Polished loading states | +2 KB | Medium |
| 3 | Hybrid Tailwind + CSS Modules | Best of both worlds | ~10 KB | Medium |
| 4 | shadcn/ui components | Accessible dialogs, toasts | +15 KB | Medium |
| 5 | Dark mode with next-themes | Full theme support | +2 KB | Medium |
| 6 | CSS variables for theming | Semantic color system | 0 KB (CSS only) | Low |

**Total bundle cost**: ~27 KB gzipped for complete styling solution

### Final Implementation: Production-Ready Product Catalog

Here's the complete, production-ready implementation with all improvements integrated:

**Project Structure**:
```
src/
├── app/
│   ├── layout.tsx              ← ThemeProvider, Toaster
│   ├── globals.css             ← Tailwind, CSS variables
│   └── products/
│       └── page.tsx            ← Product listing
├── components/
│   ├── ui/                     ← shadcn/ui components
│   │   ├── dialog.tsx
│   │   ├── toast.tsx
│   │   ├── button.tsx
│   │   └── dropdown-menu.tsx
│   ├── ProductCard.tsx         ← Hybrid styling
│   ├── ProductCard.module.css  ← Custom animations
│   ├── ProductCardSkeleton.tsx ← Loading state
│   ├── ProductGrid.tsx         ← Responsive grid
│   ├── ProductDetailDialog.tsx ← Modal with toast
│   └── ThemeToggle.tsx         ← Dark mode toggle
└── lib/
    └── products.ts             ← Data fetching
```

```tsx
// src/app/layout.tsx - Final root layout
import './globals.css';
import { ThemeProvider } from 'next-themes';
import { Toaster } from '@/components/ui/toaster';

export const metadata = {
  title: 'Product Catalog',
  description: 'Browse our products',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
          {children}
          <Toaster />
        </ThemeProvider>
      </body>
    </html>
  );
}
```

```tsx
// src/components/ProductCard.tsx - Final implementation
import Image from 'next/image';
import { Product } from '@/lib/products';
import { ProductDetailDialog } from './ProductDetailDialog';
import styles from './ProductCard.module.css';

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  return (
    <ProductDetailDialog product={product}>
      <div className="group cursor-pointer">
        <div className="overflow-hidden rounded-lg border bg-card shadow-sm transition-shadow hover:shadow-md">
          <div className={`relative aspect-square overflow-hidden bg-muted ${styles.imageContainer}`}>
            <Image
              src={product.image}
              alt={product.name}
              fill
              className="object-cover transition-transform group-hover:scale-105"
            />
            {product.isNew && (
              <span className={styles.badge}>New</span>
            )}
          </div>
          
          <div className="p-4">
            <p className="text-sm text-muted-foreground">
              {product.category}
            </p>
            <h3 className="mt-1 text-lg font-semibold text-card-foreground group-hover:text-primary">
              {product.name}
            </h3>
            <p className="mt-2 text-xl font-bold text-card-foreground">
              ${product.price.toFixed(2)}
            </p>
          </div>
        </div>
      </div>
    </ProductDetailDialog>
  );
}
```

```tsx
// src/app/products/page.tsx - Final products page
import { Suspense } from 'react';
import { getProducts } from '@/lib/products';
import { ProductGrid } from '@/components/ProductGrid';
import { ProductCardSkeleton } from '@/components/ProductCardSkeleton';
import { ThemeToggle } from '@/components/ThemeToggle';

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <div className="min-h-screen bg-background">
      <div className="mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
        <div className="mb-8 flex items-center justify-between">
          <h1 className="text-3xl font-bold text-foreground">
            Our Products
          </h1>
          <ThemeToggle />
        </div>
        
        <Suspense fallback={
          <div className="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
            {Array.from({ length: 8 }).map((_, i) => (
              <ProductCardSkeleton key={i} />
            ))}
          </div>
        }>
          <ProductGrid products={products} />
        </Suspense>
      </div>
    </div>
  );
}
```

### Decision Framework: Choosing Your Styling Approach

Use this flowchart to decide which styling approach to use:

**For standard UI components**:
- ✅ Use Tailwind utilities
- Fast development, consistent design
- Example: layouts, spacing, typography, colors

**For complex animations or pseudo-elements**:
- ✅ Use CSS Modules
- Full CSS power when needed
- Example: shimmer effects, custom badges, complex keyframes

**For accessible, interactive components**:
- ✅ Use shadcn/ui
- Production-ready, accessible, customizable
- Example: dialogs, dropdowns, toasts, forms

**For theming and dark mode**:
- ✅ Use CSS variables + next-themes
- Semantic colors, no hydration issues
- Example: theme switching, brand customization

**Hybrid approach** (recommended):
- Tailwind for 80% of styling
- CSS Modules for complex custom styles
- shadcn/ui for interactive components
- CSS variables for theming

### Lessons Learned: Styling in Next.js

**1. Start with Tailwind**
- Fastest path to good-looking UI
- Built-in responsive design
- Automatic purging keeps bundle small

**2. Add CSS Modules selectively**
- Only when Tailwind can't express what you need
- Complex animations, pseudo-elements with content
- Keep CSS Modules focused and minimal

**3. Don't reinvent accessible components**
- Use shadcn/ui for dialogs, dropdowns, toasts
- Accessibility is hard to get right
- Copy-paste approach gives you full control

**4. Plan for dark mode early**
- Easier to add dark mode variants as you build
- Use CSS variables for semantic colors
- next-themes prevents hydration issues

**5. Optimize for maintainability**
- Co-locate styles with components
- Use semantic color names (bg-card, text-foreground)
- Document custom CSS Modules
- Keep Tailwind classes readable (use Prettier plugin)

**6. Test in both themes**
- Every component should work in light and dark mode
- Check contrast ratios for accessibility
- Verify focus states are visible in both themes

**7. Monitor bundle size**
- Tailwind purges unused CSS automatically
- CSS Modules are code-split per component
- shadcn/ui only bundles what you use
- Total styling overhead should be < 50 KB gzipped

### Performance Metrics: Before and After

**Before styling**:
- Bundle size: 120 KB (Next.js + React only)
- First Contentful Paint: 0.8s
- Largest Contentful Paint: 1.2s
- Cumulative Layout Shift: 0.05

**After complete styling solution**:
- Bundle size: 147 KB (+27 KB for styling)
- First Contentful Paint: 0.9s (+0.1s)
- Largest Contentful Paint: 1.3s (+0.1s)
- Cumulative Layout Shift: 0.02 (improved, images have aspect-ratio)

**Impact**: Minimal performance cost for significant UX improvement.

### Common Pitfalls and How to Avoid Them

**Pitfall 1: Verbose className strings**
```tsx
// ❌ Hard to read
<div className="flex items-center justify-between rounded-lg border border-gray-200 bg-white p-4 shadow-sm transition-shadow hover:shadow-md dark:border-gray-700 dark:bg-gray-800">
```

**Solution**: Extract to component or use CSS Modules for complex styles
```tsx
// ✅ More readable
<div className="card-container">
  {/* Tailwind for simple utilities, CSS Module for complex pattern */}
</div>
```

**Pitfall 2: Forgetting dark mode variants**
```tsx
// ❌ Only works in light mode
<div className="bg-white text-gray-900">
```

**Solution**: Always add dark mode variants or use semantic colors
```tsx
// ✅ Works in both modes
<div className="bg-card text-card-foreground">
```

**Pitfall 3: Hydration mismatches with theme**
```tsx
// ❌ Causes hydration mismatch
const isDark = localStorage.getItem('theme') === 'dark';
return <div className={isDark ? 'dark-class' : 'light-class'}>
```

**Solution**: Use next-themes and mounted check
```tsx
// ✅ No hydration mismatch
const [mounted, setMounted] = useState(false);
useEffect(() => setMounted(true), []);
if (!mounted) return <Placeholder />;
```

**Pitfall 4: Not purging unused CSS**
```tsx
// ❌ Tailwind config missing content paths
module.exports = {
  content: [], // Empty!
}
```

**Solution**: Include all component paths
```tsx
// ✅ Purges unused CSS
module.exports = {
  content: [
    './src/**/*.{js,ts,jsx,tsx}',
  ],
}
```

**Pitfall 5: Overusing CSS Modules**
```css
/* ❌ Reinventing Tailwind utilities */
.container {
  display: flex;
  padding: 1rem;
  margin-bottom: 2rem;
}
```

**Solution**: Use Tailwind for standard utilities
```tsx
// ✅ Simpler and more maintainable
<div className="flex p-4 mb-8">
```

### The Professional React Developer's Styling Mental Model

**Think in layers**:
1. **Base layer**: Tailwind utilities for standard styling
2. **Custom layer**: CSS Modules for complex, unique styles
3. **Component layer**: shadcn/ui for interactive, accessible components
4. **Theme layer**: CSS variables for colors and theming

**Optimize for**:
- Developer experience (fast iteration)
- User experience (smooth interactions, dark mode)
- Maintainability (clear patterns, minimal custom CSS)
- Performance (small bundle, fast load)

**Remember**:
- Styling is not just aesthetics—it's UX, accessibility, and performance
- Good styling is invisible; users notice when it's missing
- Consistency matters more than perfection
- Dark mode is expected in modern applications
- Accessible components are not optional

You now have a complete, production-ready styling solution for Next.js applications. Your product catalog is responsive, accessible, themeable, and performant. The patterns you've learned apply to any Next.js project, from simple landing pages to complex web applications.
