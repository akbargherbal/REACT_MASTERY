# Chapter 9: TypeScript in Practice

## Migrating existing JavaScript code

## Migrating Existing JavaScript Code

In Chapter 8, we added TypeScript to our User Profile Dashboard from scratch. But what happens when you inherit a real codebase—thousands of lines of JavaScript, no types, and production traffic? You can't rewrite everything overnight.

This section demonstrates the **incremental migration strategy** that professional teams use: start with the most critical files, add types gradually, and let TypeScript guide you to hidden bugs.

### Reference Implementation: The Legacy Product Catalog

We're inheriting a JavaScript e-commerce component that's been in production for two years. It works—mostly. But every few months, a runtime error crashes the checkout flow because someone passed the wrong shape of data.

**Project Structure**:
```
src/
├── components/
│   ├── ProductCard.jsx          ← Our migration target
│   ├── ProductList.jsx          ← Depends on ProductCard
│   └── ShoppingCart.jsx         ← Uses product data
├── utils/
│   ├── formatPrice.js           ← Utility functions
│   └── api.js                   ← API client
└── types/
    └── product.ts               ← We'll create this
```

Here's the working JavaScript code we're starting with:

```jsx
// src/components/ProductCard.jsx
import React from 'react';
import { formatPrice } from '../utils/formatPrice';

export function ProductCard({ product, onAddToCart }) {
  const handleClick = () => {
    onAddToCart(product);
  };

  return (
    <div className="product-card">
      <img src={product.imageUrl} alt={product.name} />
      <h3>{product.name}</h3>
      <p>{product.description}</p>
      <div className="price">
        {product.onSale && (
          <span className="original-price">
            {formatPrice(product.originalPrice)}
          </span>
        )}
        <span className="current-price">
          {formatPrice(product.price)}
        </span>
      </div>
      <button onClick={handleClick}>Add to Cart</button>
      {product.inStock === false && (
        <div className="out-of-stock">Out of Stock</div>
      )}
    </div>
  );
}
```

```javascript
// src/utils/formatPrice.js
export function formatPrice(price) {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD'
  }).format(price);
}
```

```javascript
// src/utils/api.js
export async function fetchProducts() {
  const response = await fetch('/api/products');
  return response.json();
}

export async function addToCart(productId, quantity) {
  const response = await fetch('/api/cart', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ productId, quantity })
  });
  return response.json();
}
```

This code runs in production. Users can browse products and add them to their cart. But here's what happened last month:

**The Failure: Runtime Error in Production**

A product manager added a new product type to the database—a digital download with no physical inventory. The `inStock` field was `null` instead of `true` or `false`. The component rendered, but the checkout flow crashed when trying to validate inventory.

**Browser Console**:
```
TypeError: Cannot read properties of null (reading 'toString')
  at ShoppingCart.validateInventory (ShoppingCart.jsx:45)
  at ShoppingCart.handleCheckout (ShoppingCart.jsx:78)
```

**The Problem**: JavaScript allowed `null` to flow through the entire application until it hit a function that expected a boolean. No warning, no error—until a customer tried to check out.

**What We Need**: TypeScript to catch this at compile time, not runtime.

### Diagnostic Analysis: Why JavaScript Failed Us

**Browser Behavior**:
- Product card renders normally
- "Add to Cart" button works
- Checkout page loads
- Click "Complete Purchase" → White screen, error overlay

**Browser Console Output**:
```
TypeError: Cannot read properties of null (reading 'toString')
  at ShoppingCart.validateInventory (ShoppingCart.jsx:45)
  at ShoppingCart.handleCheckout (ShoppingCart.jsx:78)
  at onClick (ShoppingCart.jsx:92)
```

**React DevTools Evidence**:
- `ProductCard` component rendered successfully
- Props: `{ product: { id: "123", name: "eBook", inStock: null, ... } }`
- No warnings, no errors—React doesn't know `null` is wrong

**Let's parse this evidence**:

1. **What the user experiences**: Product appears normal, but checkout crashes with cryptic error

2. **What the console reveals**: Error happens deep in the checkout flow, far from where the bad data entered

3. **What DevTools shows**: The `null` value is visible in props, but nothing indicates it's problematic

4. **Root cause identified**: JavaScript's type system allowed invalid data to propagate through multiple components

5. **Why JavaScript can't solve this**: No compile-time checks, no editor warnings, no way to enforce data shape

6. **What we need**: TypeScript to define the expected shape of `product` and catch mismatches before deployment

### Iteration 1: The Incremental Migration Strategy

**Current state recap**: We have working JavaScript code that occasionally fails with runtime type errors.

**Migration goal**: Add TypeScript gradually without breaking existing functionality.

**The Strategy**:
1. Rename `.jsx` → `.tsx` (or `.js` → `.ts`)
2. Add `// @ts-check` to get basic checking without full migration
3. Define types for the most critical data structures
4. Fix errors one file at a time
5. Gradually tighten `tsconfig.json` settings

Let's start with the most critical piece: the product data structure.

**Step 1: Define the Product Type**

Create a new file to define what a valid product looks like:

```typescript
// src/types/product.ts

/**
 * Core product data structure used throughout the application.
 * This type represents products from our API and database.
 */
export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  imageUrl: string;
  inStock: boolean;  // ← Must be boolean, not null
  onSale?: boolean;  // ← Optional: only present when on sale
  originalPrice?: number;  // ← Optional: only present when on sale
}

/**
 * Type guard to validate unknown data is a valid Product.
 * Use this when receiving data from APIs or external sources.
 */
export function isProduct(data: unknown): data is Product {
  if (typeof data !== 'object' || data === null) {
    return false;
  }

  const obj = data as Record<string, unknown>;

  return (
    typeof obj.id === 'string' &&
    typeof obj.name === 'string' &&
    typeof obj.description === 'string' &&
    typeof obj.price === 'number' &&
    typeof obj.imageUrl === 'string' &&
    typeof obj.inStock === 'boolean' &&
    (obj.onSale === undefined || typeof obj.onSale === 'boolean') &&
    (obj.originalPrice === undefined || typeof obj.originalPrice === 'number')
  );
}
```

**Step 2: Migrate ProductCard to TypeScript**

Rename `ProductCard.jsx` → `ProductCard.tsx` and add type annotations:

```tsx
// src/components/ProductCard.tsx
import React from 'react';
import { formatPrice } from '../utils/formatPrice';
import { Product } from '../types/product';

interface ProductCardProps {
  product: Product;
  onAddToCart: (product: Product) => void;
}

export function ProductCard({ product, onAddToCart }: ProductCardProps) {
  const handleClick = () => {
    onAddToCart(product);
  };

  return (
    <div className="product-card">
      <img src={product.imageUrl} alt={product.name} />
      <h3>{product.name}</h3>
      <p>{product.description}</p>
      <div className="price">
        {product.onSale && (
          <span className="original-price">
            {formatPrice(product.originalPrice!)}
          </span>
        )}
        <span className="current-price">
          {formatPrice(product.price)}
        </span>
      </div>
      <button onClick={handleClick}>Add to Cart</button>
      {product.inStock === false && (
        <div className="out-of-stock">Out of Stock</div>
      )}
    </div>
  );
}
```

**Step 3: Attempt to Build**

Run the TypeScript compiler:

```bash
npx tsc --noEmit
```

**Terminal Output**:
```bash
src/components/ProductCard.tsx:18:25 - error TS2532:
Object is possibly 'undefined'.

18           {formatPrice(product.originalPrice!)}
                           ~~~~~~~~~~~~~~~~~~~~~

src/utils/formatPrice.js:1:1 - error TS7016:
Could not find a declaration file for module '../utils/formatPrice'.
'/src/utils/formatPrice.js' implicitly has an 'any' type.

1 import { formatPrice } from '../utils/formatPrice';
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Found 2 errors.
```

**Diagnostic Analysis: Reading TypeScript's Feedback**

**Terminal Output**: Two distinct errors

**Let's parse this evidence**:

1. **Error 1: `Object is possibly 'undefined'`**
   - Location: Line 18, `product.originalPrice`
   - Cause: We marked `originalPrice` as optional (`originalPrice?: number`)
   - TypeScript sees: "You're accessing a property that might not exist"
   - The `!` (non-null assertion) is a temporary workaround, but TypeScript still warns us

2. **Error 2: `Could not find a declaration file`**
   - Location: Import statement for `formatPrice`
   - Cause: `formatPrice.js` is still JavaScript, no type information
   - TypeScript sees: "I don't know what this function returns or what parameters it accepts"

**Root cause identified**: We're mixing TypeScript and JavaScript files, and TypeScript can't infer types across the boundary.

**What we need**: Either migrate `formatPrice.js` to TypeScript, or add type declarations for it.

### Iteration 2: Fixing the Type Errors

**Current state recap**: We've added types to `ProductCard`, but TypeScript found two issues: optional property access and untyped JavaScript imports.

**Current limitation**: Can't build because of type errors.

**Solution 1: Fix the Optional Property Access**

The `originalPrice` is only present when `onSale` is true. We need to check both conditions:

```tsx
// src/components/ProductCard.tsx (Fixed)
import React from 'react';
import { formatPrice } from '../utils/formatPrice';
import { Product } from '../types/product';

interface ProductCardProps {
  product: Product;
  onAddToCart: (product: Product) => void;
}

export function ProductCard({ product, onAddToCart }: ProductCardProps) {
  const handleClick = () => {
    onAddToCart(product);
  };

  return (
    <div className="product-card">
      <img src={product.imageUrl} alt={product.name} />
      <h3>{product.name}</h3>
      <p>{product.description}</p>
      <div className="price">
        {product.onSale && product.originalPrice !== undefined && (
          <span className="original-price">
            {formatPrice(product.originalPrice)}
          </span>
        )}
        <span className="current-price">
          {formatPrice(product.price)}
        </span>
      </div>
      <button onClick={handleClick}>Add to Cart</button>
      {product.inStock === false && (
        <div className="out-of-stock">Out of Stock</div>
      )}
    </div>
  );
}
```

**Solution 2: Migrate formatPrice to TypeScript**

Rename `formatPrice.js` → `formatPrice.ts` and add type annotations:

```typescript
// src/utils/formatPrice.ts
export function formatPrice(price: number): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD'
  }).format(price);
}
```

**Verification: Build Again**

```bash
npx tsc --noEmit
```

**Terminal Output**:
```bash
✓ No errors found
```

**Success!** The component now compiles. But more importantly, let's see what TypeScript prevents:

**Attempting to Pass Invalid Data**:

```tsx
// src/components/ProductList.tsx
import { ProductCard } from './ProductCard';

function ProductList() {
  // This would have caused a runtime error in JavaScript
  const invalidProduct = {
    id: "123",
    name: "eBook",
    description: "A digital download",
    price: 9.99,
    imageUrl: "/ebook.jpg",
    inStock: null,  // ← TypeScript error!
  };

  return (
    <ProductCard 
      product={invalidProduct}  // ← Error here
      onAddToCart={(p) => console.log(p)}
    />
  );
}
```

**Terminal Output**:
```bash
src/components/ProductList.tsx:15:7 - error TS2322:
Type 'null' is not assignable to type 'boolean'.

15       inStock: null,
         ~~~~~~~

src/components/ProductList.tsx:19:7 - error TS2322:
Type '{ id: string; name: string; ...; inStock: null; }' is not assignable to type 'Product'.
  Types of property 'inStock' are incompatible.
    Type 'null' is not assignable to type 'boolean'.

19       product={invalidProduct}
         ~~~~~~~
```

**Expected vs. Actual Improvement**:

**Before (JavaScript)**:
- Invalid data flows through the application
- Runtime error in checkout flow
- Customer sees white screen
- Error discovered in production

**After (TypeScript)**:
- Invalid data caught at compile time
- Error shown in editor with red squiggle
- Build fails before deployment
- Error discovered during development

**When to Apply This Solution**:

**What it optimizes for**: Catching data shape errors before runtime

**What it sacrifices**: Initial migration effort, learning curve

**When to choose this approach**:
- You have a codebase with frequent runtime type errors
- Multiple developers work on the same data structures
- You want editor autocomplete and refactoring support
- You're building a long-lived application

**When to avoid this approach**:
- Prototyping or throwaway code
- Very small projects (< 500 lines)
- Team strongly resists TypeScript
- Tight deadline with no time for migration

**Code characteristics**:
- Setup complexity: Medium (initial migration takes time)
- Maintenance burden: Lower (types document expected shapes)
- Performance impact: None (types are erased at runtime)

### Iteration 3: Migrating the API Client

**Current state recap**: We've migrated `ProductCard` and `formatPrice` to TypeScript, and TypeScript now catches invalid product data at compile time.

**Current limitation**: The API client (`api.js`) still returns `any`, so we lose type safety at the data boundary.

**New scenario introduction**: What happens when the API returns unexpected data?

**The Failure: API Returns Unexpected Shape**

The backend team deployed a change: they renamed `imageUrl` → `image_url` (snake_case). The API now returns:

```json
{
  "id": "123",
  "name": "Laptop",
  "image_url": "laptop.jpg",  // ← Changed from imageUrl
  "price": 999.99,
  "inStock": true
}
```

**Browser Behavior**:
- Product cards render
- Images don't load (broken image icon)
- No console errors
- No TypeScript errors

**Browser Console Output**:
```
GET http://localhost:3000/undefined 404 (Not Found)
```

**React DevTools Evidence**:
- `ProductCard` props: `{ product: { ..., imageUrl: undefined, image_url: "laptop.jpg" } }`
- The component received the wrong property name

**Diagnostic Analysis: The Type Boundary Problem**

**Let's parse this evidence**:

1. **What the user experiences**: Broken images, but no obvious error

2. **What the console reveals**: 404 error for `undefined` URL—the `imageUrl` property doesn't exist

3. **What DevTools shows**: The API returned `image_url`, but our component expects `imageUrl`

4. **Root cause identified**: TypeScript can't validate data from external sources (APIs, localStorage, user input)

5. **Why the current approach can't solve this**: The API client returns `any`, so TypeScript assumes the data is correct

6. **What we need**: Runtime validation at the API boundary to ensure data matches our types

### Solution: Type-Safe API Client with Runtime Validation

Migrate `api.js` → `api.ts` and add runtime validation:

```typescript
// src/utils/api.ts
import { Product, isProduct } from '../types/product';

/**
 * Custom error for API validation failures.
 * Includes the invalid data for debugging.
 */
export class ValidationError extends Error {
  constructor(
    message: string,
    public readonly invalidData: unknown
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

/**
 * Fetch all products from the API.
 * Validates each product matches the Product type.
 * @throws {ValidationError} if any product fails validation
 */
export async function fetchProducts(): Promise<Product[]> {
  const response = await fetch('/api/products');
  
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  const data: unknown = await response.json();

  // Validate the response is an array
  if (!Array.isArray(data)) {
    throw new ValidationError(
      'API returned non-array response',
      data
    );
  }

  // Validate each product
  const products: Product[] = [];
  for (let i = 0; i < data.length; i++) {
    const item = data[i];
    if (!isProduct(item)) {
      throw new ValidationError(
        `Invalid product at index ${i}`,
        item
      );
    }
    products.push(item);
  }

  return products;
}

/**
 * Add a product to the shopping cart.
 * @returns The updated cart data
 */
export async function addToCart(
  productId: string,
  quantity: number
): Promise<{ success: boolean; cartTotal: number }> {
  const response = await fetch('/api/cart', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ productId, quantity })
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  const data: unknown = await response.json();

  // Validate the response shape
  if (
    typeof data !== 'object' ||
    data === null ||
    typeof (data as any).success !== 'boolean' ||
    typeof (data as any).cartTotal !== 'number'
  ) {
    throw new ValidationError(
      'Invalid cart response from API',
      data
    );
  }

  return data as { success: boolean; cartTotal: number };
}
```

**Verification: Catching API Shape Mismatches**

Now when the API returns the wrong shape:

**Browser Console Output**:
```
ValidationError: Invalid product at index 0
  at fetchProducts (api.ts:45)
  at ProductList.loadProducts (ProductList.tsx:12)

Invalid data: {
  "id": "123",
  "name": "Laptop",
  "image_url": "laptop.jpg",  // ← Wrong property name
  "price": 999.99,
  "inStock": true
}
```

**Expected vs. Actual Improvement**:

**Before (Untyped API)**:
- API returns wrong shape
- Component receives `undefined` for `imageUrl`
- Broken images, no clear error
- Developer spends 30 minutes debugging

**After (Type-Safe API)**:
- API returns wrong shape
- Runtime validation catches it immediately
- Clear error message with invalid data
- Developer sees exactly what's wrong in 30 seconds

**Limitation preview**: This works great for simple types, but what about complex nested objects or third-party APIs with hundreds of fields? We'll address that in Section 9.2 with Zod.

### The Migration Journey So Far

| File | Status | Type Safety | Runtime Validation |
|------|--------|-------------|-------------------|
| `ProductCard.tsx` | ✅ Migrated | Full | N/A (props validated by caller) |
| `formatPrice.ts` | ✅ Migrated | Full | N/A (simple function) |
| `api.ts` | ✅ Migrated | Full | ✅ Manual validation |
| `ProductList.jsx` | ⏳ Pending | None | None |
| `ShoppingCart.jsx` | ⏳ Pending | None | None |

### Lessons Learned: Incremental Migration Strategy

**1. Start with data structures**: Define types for your core domain objects first (Product, User, Order, etc.)

**2. Migrate from the inside out**: Start with leaf components (ProductCard) before parent components (ProductList)

**3. Add runtime validation at boundaries**: APIs, localStorage, user input—anywhere data enters your application

**4. Use type guards for validation**: The `isProduct` function serves double duty: runtime validation and TypeScript type narrowing

**5. Don't use `any` as a crutch**: If you don't know the type, use `unknown` and validate it

**6. Leverage TypeScript's incremental mode**: You can mix `.js` and `.ts` files during migration—TypeScript will check what it can

**Next**: In Section 9.2, we'll replace our manual validation with Zod, a runtime validation library that generates TypeScript types automatically.

## Type-safe API clients

## Type-Safe API Clients

In Section 9.1, we manually wrote runtime validation for our API responses. It worked, but it was tedious and error-prone. For every new field, we had to:

1. Add it to the TypeScript interface
2. Add a validation check in the type guard
3. Keep both in sync manually

This doesn't scale. Professional teams use **schema validation libraries** that define the shape once and generate both runtime validation and TypeScript types automatically.

### The Problem with Manual Validation

Let's see what happens when our Product type grows:

```typescript
// src/types/product.ts (Growing complexity)
export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  imageUrl: string;
  inStock: boolean;
  onSale?: boolean;
  originalPrice?: number;
  // New fields added over time
  category: string;
  tags: string[];
  rating: number;
  reviewCount: number;
  dimensions?: {
    width: number;
    height: number;
    depth: number;
    weight: number;
  };
  variants?: Array<{
    id: string;
    name: string;
    price: number;
    inStock: boolean;
  }>;
}

// The type guard becomes a nightmare
export function isProduct(data: unknown): data is Product {
  if (typeof data !== 'object' || data === null) return false;
  const obj = data as Record<string, unknown>;

  // Basic fields
  if (typeof obj.id !== 'string') return false;
  if (typeof obj.name !== 'string') return false;
  if (typeof obj.description !== 'string') return false;
  if (typeof obj.price !== 'number') return false;
  if (typeof obj.imageUrl !== 'string') return false;
  if (typeof obj.inStock !== 'boolean') return false;
  
  // Optional fields
  if (obj.onSale !== undefined && typeof obj.onSale !== 'boolean') return false;
  if (obj.originalPrice !== undefined && typeof obj.originalPrice !== 'number') return false;

  // New fields
  if (typeof obj.category !== 'string') return false;
  if (!Array.isArray(obj.tags)) return false;
  if (!obj.tags.every(tag => typeof tag === 'string')) return false;
  if (typeof obj.rating !== 'number') return false;
  if (typeof obj.reviewCount !== 'number') return false;

  // Nested optional object
  if (obj.dimensions !== undefined) {
    if (typeof obj.dimensions !== 'object' || obj.dimensions === null) return false;
    const dims = obj.dimensions as Record<string, unknown>;
    if (typeof dims.width !== 'number') return false;
    if (typeof dims.height !== 'number') return false;
    if (typeof dims.depth !== 'number') return false;
    if (typeof dims.weight !== 'number') return false;
  }

  // Array of nested objects
  if (obj.variants !== undefined) {
    if (!Array.isArray(obj.variants)) return false;
    for (const variant of obj.variants) {
      if (typeof variant !== 'object' || variant === null) return false;
      const v = variant as Record<string, unknown>;
      if (typeof v.id !== 'string') return false;
      if (typeof v.name !== 'string') return false;
      if (typeof v.price !== 'number') return false;
      if (typeof v.inStock !== 'boolean') return false;
    }
  }

  return true;
}
```

**The Failure: Validation and Types Drift Apart**

A developer adds a new field to the `Product` interface but forgets to update the type guard:

```typescript
export interface Product {
  // ... existing fields ...
  shippingWeight?: number;  // ← Added to interface
}

// Type guard NOT updated
export function isProduct(data: unknown): data is Product {
  // ... validation doesn't check shippingWeight ...
}
```

**Browser Behavior**:
- API returns product with `shippingWeight: "5kg"` (string instead of number)
- Type guard passes (doesn't check this field)
- TypeScript thinks it's a number
- Shipping calculation crashes: `shippingWeight.toFixed(2)` fails

**Browser Console Output**:
```
TypeError: shippingWeight.toFixed is not a function
  at calculateShipping (shipping.ts:23)
  at ShoppingCart.render (ShoppingCart.tsx:45)
```

**Diagnostic Analysis: The Sync Problem**

**Let's parse this evidence**:

1. **What the user experiences**: Checkout crashes when calculating shipping

2. **What the console reveals**: Trying to call `.toFixed()` on a string

3. **What DevTools shows**: `shippingWeight` is `"5kg"` (string), but TypeScript thinks it's a number

4. **Root cause identified**: Type definition and runtime validation are separate, manually maintained, and drifted apart

5. **Why manual validation can't solve this**: Human error—it's too easy to update one without the other

6. **What we need**: A single source of truth that generates both TypeScript types and runtime validation

### Iteration 1: Introducing Zod

**Zod** is a TypeScript-first schema validation library. You define the shape once, and Zod generates:
- Runtime validation
- TypeScript types
- Detailed error messages

Install Zod:

```bash
npm install zod
```

Let's rewrite our Product type using Zod:

```typescript
// src/types/product.ts (Zod version)
import { z } from 'zod';

/**
 * Product schema - single source of truth for shape and validation.
 * TypeScript types are inferred automatically from this schema.
 */
export const ProductSchema = z.object({
  id: z.string(),
  name: z.string(),
  description: z.string(),
  price: z.number().positive(),
  imageUrl: z.string().url(),
  inStock: z.boolean(),
  onSale: z.boolean().optional(),
  originalPrice: z.number().positive().optional(),
  category: z.string(),
  tags: z.array(z.string()),
  rating: z.number().min(0).max(5),
  reviewCount: z.number().int().nonnegative(),
  dimensions: z.object({
    width: z.number().positive(),
    height: z.number().positive(),
    depth: z.number().positive(),
    weight: z.number().positive(),
  }).optional(),
  variants: z.array(z.object({
    id: z.string(),
    name: z.string(),
    price: z.number().positive(),
    inStock: z.boolean(),
  })).optional(),
});

/**
 * TypeScript type inferred from the schema.
 * This is automatically kept in sync with the schema.
 */
export type Product = z.infer<typeof ProductSchema>;

/**
 * Validate unknown data against the Product schema.
 * Returns the validated data or throws a detailed error.
 */
export function validateProduct(data: unknown): Product {
  return ProductSchema.parse(data);
}

/**
 * Safe validation that returns a result object instead of throwing.
 * Use this when you want to handle validation errors gracefully.
 */
export function safeValidateProduct(data: unknown): 
  { success: true; data: Product } | { success: false; error: z.ZodError } {
  const result = ProductSchema.safeParse(data);
  return result;
}
```

**Key Improvements**:

1. **Single source of truth**: The schema defines both shape and validation
2. **Type inference**: `type Product = z.infer<typeof ProductSchema>` automatically generates the TypeScript type
3. **Built-in validation**: `.positive()`, `.url()`, `.min()`, `.max()` provide rich validation
4. **Detailed errors**: Zod tells you exactly what's wrong and where

Now let's update the API client to use Zod:

```typescript
// src/utils/api.ts (Zod version)
import { z } from 'zod';
import { Product, ProductSchema } from '../types/product';

/**
 * Fetch all products from the API.
 * Validates each product using Zod schema.
 */
export async function fetchProducts(): Promise<Product[]> {
  const response = await fetch('/api/products');
  
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  const data: unknown = await response.json();

  // Validate the entire response as an array of products
  const ProductArraySchema = z.array(ProductSchema);
  
  try {
    return ProductArraySchema.parse(data);
  } catch (error) {
    if (error instanceof z.ZodError) {
      console.error('Product validation failed:', error.format());
      throw new Error(`Invalid product data: ${error.message}`);
    }
    throw error;
  }
}

/**
 * Cart response schema
 */
const CartResponseSchema = z.object({
  success: z.boolean(),
  cartTotal: z.number().nonnegative(),
});

export async function addToCart(
  productId: string,
  quantity: number
): Promise<z.infer<typeof CartResponseSchema>> {
  const response = await fetch('/api/cart', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ productId, quantity })
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  const data: unknown = await response.json();
  return CartResponseSchema.parse(data);
}
```

**Verification: Catching Invalid Data with Detailed Errors**

When the API returns invalid data:

**Scenario 1: Wrong Type**

API returns:
```json
{
  "id": "123",
  "name": "Laptop",
  "price": "999.99",  // ← String instead of number
  "imageUrl": "laptop.jpg",
  "inStock": true,
  "category": "electronics",
  "tags": ["laptop", "computer"],
  "rating": 4.5,
  "reviewCount": 42
}
```

**Browser Console Output**:
```
Product validation failed: {
  price: {
    _errors: ['Expected number, received string']
  }
}

Error: Invalid product data: [
  {
    "code": "invalid_type",
    "expected": "number",
    "received": "string",
    "path": ["price"],
    "message": "Expected number, received string"
  }
]
```

**Scenario 2: Missing Required Field**

API returns:
```json
{
  "id": "123",
  "name": "Laptop",
  "price": 999.99,
  // ← Missing imageUrl
  "inStock": true,
  "category": "electronics",
  "tags": ["laptop"],
  "rating": 4.5,
  "reviewCount": 42
}
```

**Browser Console Output**:
```
Product validation failed: {
  imageUrl: {
    _errors: ['Required']
  }
}

Error: Invalid product data: [
  {
    "code": "invalid_type",
    "expected": "string",
    "received": "undefined",
    "path": ["imageUrl"],
    "message": "Required"
  }
]
```

**Scenario 3: Invalid URL Format**

API returns:
```json
{
  "id": "123",
  "name": "Laptop",
  "price": 999.99,
  "imageUrl": "not-a-url",  // ← Invalid URL
  "inStock": true,
  "category": "electronics",
  "tags": ["laptop"],
  "rating": 4.5,
  "reviewCount": 42
}
```

**Browser Console Output**:
```
Product validation failed: {
  imageUrl: {
    _errors: ['Invalid url']
  }
}
```

**Expected vs. Actual Improvement**:

**Before (Manual Validation)**:
- 50+ lines of validation code
- Easy to forget fields
- Generic error messages
- Type and validation can drift apart

**After (Zod)**:
- 20 lines of schema definition
- Impossible to forget fields (schema is the source of truth)
- Detailed, structured error messages
- Type and validation always in sync

### Iteration 2: Advanced Zod Patterns

**Current state recap**: We're using Zod for basic validation, and it's already better than manual validation.

**Current limitation**: Real-world APIs have more complex requirements—transformations, conditional fields, custom validation.

**New scenario introduction**: What if we need to:
- Transform API data (e.g., convert cents to dollars)
- Validate relationships between fields (e.g., `originalPrice` must be greater than `price` when `onSale` is true)
- Handle API versioning (different shapes for different API versions)

Let's add these capabilities:

```typescript
// src/types/product.ts (Advanced Zod patterns)
import { z } from 'zod';

/**
 * Product schema with transformations and custom validation
 */
export const ProductSchema = z.object({
  id: z.string(),
  name: z.string().min(1, "Product name cannot be empty"),
  description: z.string(),
  
  // Transform: API returns price in cents, we want dollars
  price: z.number().int().positive()
    .transform(cents => cents / 100),
  
  imageUrl: z.string().url(),
  inStock: z.boolean(),
  onSale: z.boolean().optional(),
  
  // Transform: API returns price in cents
  originalPrice: z.number().int().positive()
    .transform(cents => cents / 100)
    .optional(),
  
  category: z.string(),
  tags: z.array(z.string()),
  
  // Constrain rating to valid range
  rating: z.number().min(0).max(5),
  reviewCount: z.number().int().nonnegative(),
  
  dimensions: z.object({
    width: z.number().positive(),
    height: z.number().positive(),
    depth: z.number().positive(),
    weight: z.number().positive(),
  }).optional(),
  
  variants: z.array(z.object({
    id: z.string(),
    name: z.string(),
    price: z.number().int().positive()
      .transform(cents => cents / 100),
    inStock: z.boolean(),
  })).optional(),
})
  // Custom validation: if onSale is true, originalPrice must exist and be greater than price
  .refine(
    (data) => {
      if (data.onSale) {
        return data.originalPrice !== undefined && data.originalPrice > data.price;
      }
      return true;
    },
    {
      message: "When onSale is true, originalPrice must be greater than price",
      path: ["originalPrice"],
    }
  );

export type Product = z.infer<typeof ProductSchema>;

/**
 * API v1 schema - older API version with different field names
 */
export const ProductSchemaV1 = z.object({
  product_id: z.string(),
  product_name: z.string(),
  price_cents: z.number().int().positive(),
  image_url: z.string().url(),
  in_stock: z.boolean(),
  // ... other fields
})
  // Transform to match our internal Product type
  .transform((data) => ({
    id: data.product_id,
    name: data.product_name,
    price: data.price_cents / 100,
    imageUrl: data.image_url,
    inStock: data.in_stock,
    // ... map other fields
  }));

/**
 * Union type for handling multiple API versions
 */
export const ProductSchemaAnyVersion = z.union([
  ProductSchema,
  ProductSchemaV1,
]);
```

**Verification: Advanced Validation in Action**

**Scenario 1: Price Transformation**

API returns:
```json
{
  "id": "123",
  "name": "Laptop",
  "price": 99999,  // ← 999.99 dollars in cents
  "imageUrl": "https://example.com/laptop.jpg",
  "inStock": true,
  "category": "electronics",
  "tags": ["laptop"],
  "rating": 4.5,
  "reviewCount": 42
}
```

After validation:
```typescript
const product = ProductSchema.parse(apiData);
console.log(product.price);  // 999.99 (transformed from cents)
```

**Scenario 2: Custom Validation Failure**

API returns:
```json
{
  "id": "123",
  "name": "Laptop",
  "price": 99999,
  "imageUrl": "https://example.com/laptop.jpg",
  "inStock": true,
  "onSale": true,
  "originalPrice": 99999,  // ← Same as price, should be higher
  "category": "electronics",
  "tags": ["laptop"],
  "rating": 4.5,
  "reviewCount": 42
}
```

**Browser Console Output**:
```
Product validation failed: {
  originalPrice: {
    _errors: ['When onSale is true, originalPrice must be greater than price']
  }
}
```

**Scenario 3: API Version Handling**

Old API returns:
```json
{
  "product_id": "123",
  "product_name": "Laptop",
  "price_cents": 99999,
  "image_url": "https://example.com/laptop.jpg",
  "in_stock": true
}
```

```typescript
const product = ProductSchemaAnyVersion.parse(oldApiData);
// Automatically transformed to match Product type
console.log(product.id);        // "123"
console.log(product.name);      // "Laptop"
console.log(product.price);     // 999.99
console.log(product.imageUrl);  // "https://example.com/laptop.jpg"
```

### Iteration 3: Type-Safe API Client with Error Handling

**Current state recap**: We have powerful Zod schemas with transformations and custom validation.

**Current limitation**: When validation fails, we throw errors. But in production, we want graceful degradation—show cached data, retry, or display a user-friendly error.

**Solution: Structured Error Handling**

```typescript
// src/utils/api.ts (Production-ready version)
import { z } from 'zod';
import { Product, ProductSchema } from '../types/product';

/**
 * API error types for structured error handling
 */
export class NetworkError extends Error {
  constructor(
    message: string,
    public readonly status: number,
    public readonly statusText: string
  ) {
    super(message);
    this.name = 'NetworkError';
  }
}

export class ValidationError extends Error {
  constructor(
    message: string,
    public readonly zodError: z.ZodError
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

/**
 * Result type for operations that can fail
 */
export type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

/**
 * Fetch products with structured error handling.
 * Returns a Result type instead of throwing.
 */
export async function fetchProducts(): Promise<Result<Product[], NetworkError | ValidationError>> {
  try {
    const response = await fetch('/api/products');
    
    if (!response.ok) {
      return {
        success: false,
        error: new NetworkError(
          `Failed to fetch products`,
          response.status,
          response.statusText
        ),
      };
    }

    const data: unknown = await response.json();
    const ProductArraySchema = z.array(ProductSchema);
    
    const result = ProductArraySchema.safeParse(data);
    
    if (!result.success) {
      return {
        success: false,
        error: new ValidationError(
          'Invalid product data from API',
          result.error
        ),
      };
    }

    return { success: true, data: result.data };
    
  } catch (error) {
    // Network error (fetch failed)
    return {
      success: false,
      error: new NetworkError(
        error instanceof Error ? error.message : 'Unknown network error',
        0,
        'Network request failed'
      ),
    };
  }
}

/**
 * Add to cart with structured error handling
 */
const CartResponseSchema = z.object({
  success: z.boolean(),
  cartTotal: z.number().nonnegative(),
  itemCount: z.number().int().nonnegative(),
});

type CartResponse = z.infer<typeof CartResponseSchema>;

export async function addToCart(
  productId: string,
  quantity: number
): Promise<Result<CartResponse, NetworkError | ValidationError>> {
  try {
    const response = await fetch('/api/cart', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ productId, quantity })
    });

    if (!response.ok) {
      return {
        success: false,
        error: new NetworkError(
          'Failed to add to cart',
          response.status,
          response.statusText
        ),
      };
    }

    const data: unknown = await response.json();
    const result = CartResponseSchema.safeParse(data);

    if (!result.success) {
      return {
        success: false,
        error: new ValidationError(
          'Invalid cart response from API',
          result.error
        ),
      };
    }

    return { success: true, data: result.data };
    
  } catch (error) {
    return {
      success: false,
      error: new NetworkError(
        error instanceof Error ? error.message : 'Unknown error',
        0,
        'Request failed'
      ),
    };
  }
}
```

**Using the Type-Safe API Client in Components**

```tsx
// src/components/ProductList.tsx
import React, { useEffect, useState } from 'react';
import { fetchProducts, NetworkError, ValidationError } from '../utils/api';
import { Product } from '../types/product';
import { ProductCard } from './ProductCard';

export function ProductList() {
  const [products, setProducts] = useState<Product[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function loadProducts() {
      setIsLoading(true);
      setError(null);

      const result = await fetchProducts();

      if (result.success) {
        setProducts(result.data);
      } else {
        // Handle different error types
        if (result.error instanceof NetworkError) {
          if (result.error.status === 404) {
            setError('Products not found. Please try again later.');
          } else if (result.error.status >= 500) {
            setError('Server error. Please try again later.');
          } else {
            setError('Failed to load products. Please check your connection.');
          }
        } else if (result.error instanceof ValidationError) {
          // Log detailed validation errors for debugging
          console.error('Validation error:', result.error.zodError.format());
          setError('Received invalid data from server. Please contact support.');
        } else {
          setError('An unexpected error occurred.');
        }
      }

      setIsLoading(false);
    }

    loadProducts();
  }, []);

  if (isLoading) {
    return <div className="loading">Loading products...</div>;
  }

  if (error) {
    return (
      <div className="error">
        <p>{error}</p>
        <button onClick={() => window.location.reload()}>
          Retry
        </button>
      </div>
    );
  }

  return (
    <div className="product-list">
      {products.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          onAddToCart={(p) => console.log('Add to cart:', p)}
        />
      ))}
    </div>
  );
}
```

**Expected vs. Actual Improvement**:

**Before (Throwing Errors)**:
- Errors crash the component
- Generic error messages
- No way to distinguish network vs. validation errors
- Hard to implement retry logic

**After (Result Type)**:
- Errors are values, not exceptions
- Specific error messages for different failure modes
- Type-safe error handling
- Easy to implement retry, fallback, or caching

**When to Apply This Solution**:

**What it optimizes for**: Robust error handling, user experience, debuggability

**What it sacrifices**: More verbose code, need to handle Result type everywhere

**When to choose this approach**:
- Production applications with real users
- APIs you don't control (third-party, microservices)
- Need to distinguish different error types
- Want to implement retry logic or fallbacks

**When to avoid this approach**:
- Internal tools with controlled APIs
- Prototypes where errors are acceptable
- Simple CRUD apps with minimal error handling needs

**Code characteristics**:
- Setup complexity: Medium (need to define error types and Result type)
- Maintenance burden: Lower (errors are explicit and type-safe)
- Performance impact: None (just different error handling pattern)

### The Type-Safe API Journey

| Iteration | Validation Approach | Type Safety | Error Handling | Lines of Code |
|-----------|-------------------|-------------|----------------|---------------|
| 0 | None | None | Crashes | 10 |
| 1 | Manual type guards | Partial | Throws | 50+ |
| 2 | Zod schemas | Full | Throws | 20 |
| 3 | Zod + Result type | Full | Structured | 30 |

### Lessons Learned: Type-Safe API Clients

**1. Use schema validation libraries**: Don't write manual type guards—use Zod, Yup, or io-ts

**2. Single source of truth**: Define the schema once, infer TypeScript types from it

**3. Validate at boundaries**: APIs, localStorage, user input—anywhere data enters your application

**4. Transform data early**: Convert API shapes to your internal types at the boundary

**5. Handle errors as values**: Use Result types instead of throwing for better error handling

**6. Provide detailed errors**: Zod's error messages are excellent for debugging—log them in development

**7. Version your schemas**: Use union types to handle multiple API versions gracefully

**Next**: In Section 9.3, we'll tackle third-party libraries that don't have TypeScript types.

## Handling third-party library types

## Handling Third-Party Library Types

You've built a type-safe application. Your components are typed, your API client validates data, and TypeScript catches bugs before they reach production. Then you install a third-party library, and suddenly:

```bash
npm install awesome-charts
```

**Terminal Output**:
```bash
Could not find a declaration file for module 'awesome-charts'.
'/node_modules/awesome-charts/index.js' implicitly has an 'any' type.

Try `npm i --save-dev @types/awesome-charts` if it exists or add a new
declaration (.d.ts) file containing `declare module 'awesome-charts';`
```

Your type safety just hit a wall. The library works at runtime, but TypeScript doesn't know what types it exports.

### The Three Scenarios

When you install a third-party library, you'll encounter one of three situations:

1. **Built-in types**: The library includes TypeScript definitions (`.d.ts` files)
2. **DefinitelyTyped**: Community-maintained types available via `@types/package-name`
3. **No types**: No types exist—you need to write them yourself

Let's handle each scenario with a concrete example.

### Reference Implementation: Analytics Dashboard

We're building an analytics dashboard that needs:
- A charting library (no built-in types)
- A date manipulation library (has DefinitelyTyped types)
- A custom utility library (we'll write types)

**Project Structure**:
```
src/
├── components/
│   ├── AnalyticsDashboard.tsx   ← Our main component
│   ├── SalesChart.tsx            ← Uses charting library
│   └── DateRangePicker.tsx       ← Uses date library
├── utils/
│   └── analytics.js              ← Legacy utility (no types)
└── types/
    ├── awesome-charts.d.ts       ← We'll create this
    └── analytics.d.ts            ← We'll create this
```

### Scenario 1: Library with No Types

**The Failure: TypeScript Can't Infer Anything**

We install a charting library:

```bash
npm install awesome-charts
```

Try to use it:

```tsx
// src/components/SalesChart.tsx (Attempt 1)
import React from 'react';
import { LineChart } from 'awesome-charts';  // ← Error

interface SalesChartProps {
  data: Array<{ date: string; sales: number }>;
}

export function SalesChart({ data }: SalesChartProps) {
  return (
    <LineChart
      data={data}
      xKey="date"
      yKey="sales"
      width={600}
      height={400}
    />
  );
}
```

**Terminal Output**:
```bash
src/components/SalesChart.tsx:2:10 - error TS7016:
Could not find a declaration file for module 'awesome-charts'.
'/node_modules/awesome-charts/index.js' implicitly has an 'any' type.

2 import { LineChart } from 'awesome-charts';
           ~~~~~~~~~~

Try `npm i --save-dev @types/awesome-charts` if it exists or add a new
declaration (.d.ts) file containing `declare module 'awesome-charts';`
```

**Diagnostic Analysis: The Missing Types Problem**

**Let's parse this evidence**:

1. **What TypeScript sees**: A JavaScript file with no type information

2. **What the error reveals**: TypeScript defaults to `any` for untyped imports, which defeats the purpose of TypeScript

3. **Root cause identified**: The library author didn't include TypeScript definitions

4. **Why we can't ignore this**: Using `any` types means no autocomplete, no type checking, no refactoring support

5. **What we need**: Type definitions that describe the library's API

### Solution: Writing Declaration Files

**Step 1: Check if @types exists**

First, try installing community types:

```bash
npm install --save-dev @types/awesome-charts
```

If this fails (package not found), you need to write types yourself.

**Step 2: Create a declaration file**

Create `src/types/awesome-charts.d.ts`:

```typescript
// src/types/awesome-charts.d.ts

/**
 * Type definitions for awesome-charts
 * Project: https://github.com/example/awesome-charts
 * Definitions by: Your Name
 */

declare module 'awesome-charts' {
  import { ReactElement } from 'react';

  /**
   * Common props shared by all chart components
   */
  interface BaseChartProps {
    /** Chart width in pixels */
    width?: number;
    /** Chart height in pixels */
    height?: number;
    /** CSS class name */
    className?: string;
    /** Chart title */
    title?: string;
  }

  /**
   * Props for LineChart component
   */
  interface LineChartProps<T = any> extends BaseChartProps {
    /** Array of data points */
    data: T[];
    /** Key in data object for x-axis values */
    xKey: keyof T;
    /** Key in data object for y-axis values */
    yKey: keyof T;
    /** Line color (hex or CSS color name) */
    color?: string;
    /** Show data point markers */
    showMarkers?: boolean;
    /** Smooth line curves */
    smooth?: boolean;
  }

  /**
   * Line chart component
   */
  export function LineChart<T = any>(props: LineChartProps<T>): ReactElement;

  /**
   * Props for BarChart component
   */
  interface BarChartProps<T = any> extends BaseChartProps {
    data: T[];
    xKey: keyof T;
    yKey: keyof T;
    /** Bar color */
    color?: string;
    /** Horizontal bars instead of vertical */
    horizontal?: boolean;
  }

  /**
   * Bar chart component
   */
  export function BarChart<T = any>(props: BarChartProps<T>): ReactElement;

  /**
   * Props for PieChart component
   */
  interface PieChartProps<T = any> extends BaseChartProps {
    data: T[];
    /** Key for slice labels */
    labelKey: keyof T;
    /** Key for slice values */
    valueKey: keyof T;
    /** Show percentage labels */
    showPercentages?: boolean;
  }

  /**
   * Pie chart component
   */
  export function PieChart<T = any>(props: PieChartProps<T>): ReactElement;
}
```

**Step 3: Configure TypeScript to find the types**

Update `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "allowImportingTsExtensions": true,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    
    // Tell TypeScript where to find custom type definitions
    "typeRoots": ["./node_modules/@types", "./src/types"],
    "types": []
  },
  "include": ["src"]
}
```

**Verification: Types Now Work**

Now the component compiles with full type safety:

```tsx
// src/components/SalesChart.tsx (Working version)
import React from 'react';
import { LineChart } from 'awesome-charts';

interface SalesData {
  date: string;
  sales: number;
}

interface SalesChartProps {
  data: SalesData[];
}

export function SalesChart({ data }: SalesChartProps) {
  return (
    <LineChart<SalesData>
      data={data}
      xKey="date"        // ← TypeScript knows this must be a key of SalesData
      yKey="sales"       // ← TypeScript knows this must be a key of SalesData
      width={600}
      height={400}
      color="#3b82f6"
      showMarkers={true}
      smooth={true}
    />
  );
}
```

**Expected vs. Actual Improvement**:

**Before (No Types)**:
- No autocomplete for props
- No error if you pass wrong prop names
- No type checking for data structure
- Runtime errors from typos

**After (Custom Types)**:
- Full autocomplete in editor
- TypeScript catches typos: `xKey="datte"` → error
- Type-safe data access: `xKey` must be a key of `SalesData`
- Compile-time errors instead of runtime crashes

**Attempting to Use Wrong Props**:

```tsx
// This now produces TypeScript errors:
<LineChart<SalesData>
  data={data}
  xKey="datte"        // ← Error: "datte" is not a key of SalesData
  yKey="revenue"      // ← Error: "revenue" is not a key of SalesData
  invalidProp={true}  // ← Error: Property 'invalidProp' does not exist
/>
```

**Terminal Output**:
```bash
src/components/SalesChart.tsx:12:3 - error TS2322:
Type '"datte"' is not assignable to type '"date" | "sales"'.

12   xKey="datte"
     ~~~~

src/components/SalesChart.tsx:13:3 - error TS2322:
Type '"revenue"' is not assignable to type '"date" | "sales"'.

13   yKey="revenue"
     ~~~~

src/components/SalesChart.tsx:14:3 - error TS2353:
Object literal may only specify known properties, and 'invalidProp' does not exist in type 'LineChartProps<SalesData>'.

14   invalidProp={true}
     ~~~~~~~~~~~
```

### Scenario 2: Library with DefinitelyTyped Types

**The Success Story: date-fns**

Many popular libraries have community-maintained types on DefinitelyTyped. Let's use `date-fns` for date manipulation:

```bash
npm install date-fns
npm install --save-dev @types/date-fns
```

**No additional work needed!** TypeScript automatically finds the types:

```tsx
// src/components/DateRangePicker.tsx
import React, { useState } from 'react';
import { format, subDays, startOfWeek, endOfWeek } from 'date-fns';

interface DateRange {
  start: Date;
  end: Date;
}

export function DateRangePicker() {
  const [range, setRange] = useState<DateRange>({
    start: subDays(new Date(), 7),
    end: new Date(),
  });

  const presets = {
    'Last 7 days': {
      start: subDays(new Date(), 7),
      end: new Date(),
    },
    'This week': {
      start: startOfWeek(new Date()),
      end: endOfWeek(new Date()),
    },
  };

  return (
    <div className="date-range-picker">
      <div className="current-range">
        {format(range.start, 'MMM d, yyyy')} - {format(range.end, 'MMM d, yyyy')}
      </div>
      <div className="presets">
        {Object.entries(presets).map(([label, preset]) => (
          <button key={label} onClick={() => setRange(preset)}>
            {label}
          </button>
        ))}
      </div>
    </div>
  );
}
```

**Full type safety with zero effort**:
- `format()` autocompletes format strings
- `subDays()` requires a number, not a string
- `Date` objects are properly typed throughout

### Scenario 3: Legacy Internal Code

**The Failure: Your Own Untyped Code**

You have a legacy utility file that other developers wrote in JavaScript:

```javascript
// src/utils/analytics.js (Legacy code)

/**
 * Calculate percentage change between two values
 */
export function calculateChange(current, previous) {
  if (previous === 0) return 0;
  return ((current - previous) / previous) * 100;
}

/**
 * Format a number as a percentage
 */
export function formatPercentage(value, decimals = 1) {
  return `${value.toFixed(decimals)}%`;
}

/**
 * Group data by time period
 */
export function groupByPeriod(data, period) {
  const groups = {};
  
  data.forEach(item => {
    const key = period === 'day' 
      ? item.date.split('T')[0]
      : item.date.substring(0, 7);
    
    if (!groups[key]) {
      groups[key] = [];
    }
    groups[key].push(item);
  });
  
  return groups;
}

/**
 * Calculate moving average
 */
export function movingAverage(data, windowSize) {
  const result = [];
  
  for (let i = 0; i < data.length; i++) {
    const start = Math.max(0, i - windowSize + 1);
    const window = data.slice(start, i + 1);
    const sum = window.reduce((acc, val) => acc + val, 0);
    result.push(sum / window.length);
  }
  
  return result;
}
```

**Attempting to Use It**:

```tsx
// src/components/AnalyticsDashboard.tsx (Attempt 1)
import React from 'react';
import { calculateChange, formatPercentage } from '../utils/analytics';

interface AnalyticsDashboardProps {
  currentSales: number;
  previousSales: number;
}

export function AnalyticsDashboard({ currentSales, previousSales }: AnalyticsDashboardProps) {
  const change = calculateChange(currentSales, previousSales);
  const formatted = formatPercentage(change);

  return (
    <div className="analytics-dashboard">
      <div className="metric">
        <span className="label">Sales Change</span>
        <span className="value">{formatted}</span>
      </div>
    </div>
  );
}
```

**Terminal Output**:
```bash
src/components/AnalyticsDashboard.tsx:2:10 - error TS7016:
Could not find a declaration file for module '../utils/analytics'.
'/src/utils/analytics.js' implicitly has an 'any' type.

2 import { calculateChange, formatPercentage } from '../utils/analytics';
           ~~~~~~~~~~~~~~~
```

**Solution: Add Type Declarations for Legacy Code**

Create `src/types/analytics.d.ts`:

```typescript
// src/types/analytics.d.ts

/**
 * Type definitions for legacy analytics utilities
 */
declare module '../utils/analytics' {
  /**
   * Calculate percentage change between two values.
   * Returns 0 if previous value is 0 to avoid division by zero.
   * 
   * @param current - Current value
   * @param previous - Previous value
   * @returns Percentage change (e.g., 25 for 25% increase)
   */
  export function calculateChange(current: number, previous: number): number;

  /**
   * Format a number as a percentage string.
   * 
   * @param value - Numeric value to format
   * @param decimals - Number of decimal places (default: 1)
   * @returns Formatted percentage string (e.g., "25.5%")
   */
  export function formatPercentage(value: number, decimals?: number): string;

  /**
   * Time period for grouping data
   */
  export type TimePeriod = 'day' | 'month';

  /**
   * Data point with date and value
   */
  export interface DataPoint {
    date: string;
    [key: string]: any;
  }

  /**
   * Group data points by time period.
   * 
   * @param data - Array of data points with date strings
   * @param period - Time period to group by ('day' or 'month')
   * @returns Object mapping period keys to arrays of data points
   */
  export function groupByPeriod(
    data: DataPoint[],
    period: TimePeriod
  ): Record<string, DataPoint[]>;

  /**
   * Calculate moving average over a sliding window.
   * 
   * @param data - Array of numeric values
   * @param windowSize - Size of the moving window
   * @returns Array of averaged values (same length as input)
   */
  export function movingAverage(data: number[], windowSize: number): number[];
}
```

**Verification: Legacy Code Now Type-Safe**

The component now has full type safety:

```tsx
// src/components/AnalyticsDashboard.tsx (Working version)
import React from 'react';
import { 
  calculateChange, 
  formatPercentage,
  groupByPeriod,
  movingAverage,
  type DataPoint,
  type TimePeriod
} from '../utils/analytics';

interface AnalyticsDashboardProps {
  currentSales: number;
  previousSales: number;
  historicalData: DataPoint[];
}

export function AnalyticsDashboard({ 
  currentSales, 
  previousSales,
  historicalData 
}: AnalyticsDashboardProps) {
  const change = calculateChange(currentSales, previousSales);
  const formatted = formatPercentage(change, 2);  // ← TypeScript knows decimals is optional

  // Group data by month
  const monthlyData = groupByPeriod(historicalData, 'month');

  // Calculate 7-day moving average
  const salesValues = historicalData.map(d => d.sales);
  const smoothed = movingAverage(salesValues, 7);

  return (
    <div className="analytics-dashboard">
      <div className="metric">
        <span className="label">Sales Change</span>
        <span className="value">{formatted}</span>
      </div>
      {/* ... render charts with smoothed data ... */}
    </div>
  );
}
```

**Attempting to Use Wrong Types**:

```tsx
// These now produce TypeScript errors:
calculateChange("100", "90");           // ← Error: Expected number, got string
formatPercentage(change, "2");          // ← Error: Expected number, got string
groupByPeriod(historicalData, 'year');  // ← Error: 'year' not in TimePeriod
movingAverage(historicalData, 7);       // ← Error: Expected number[], got DataPoint[]
```

**Terminal Output**:
```bash
src/components/AnalyticsDashboard.tsx:15:18 - error TS2345:
Argument of type 'string' is not assignable to parameter of type 'number'.

15   calculateChange("100", "90");
                      ~~~~~

src/components/AnalyticsDashboard.tsx:16:30 - error TS2345:
Argument of type 'string' is not assignable to parameter of type 'number | undefined'.

16   formatPercentage(change, "2");
                                ~~~

src/components/AnalyticsDashboard.tsx:17:38 - error TS2345:
Argument of type '"year"' is not assignable to parameter of type 'TimePeriod'.

17   groupByPeriod(historicalData, 'year');
                                    ~~~~~~

src/components/AnalyticsDashboard.tsx:18:17 - error TS2345:
Argument of type 'DataPoint[]' is not assignable to parameter of type 'number[]'.

18   movingAverage(historicalData, 7);
                   ~~~~~~~~~~~~~~
```

### When to Apply Each Solution

| Scenario | Solution | Effort | Maintenance |
|----------|----------|--------|-------------|
| Library with built-in types | None needed | None | None |
| Library on DefinitelyTyped | `npm i @types/package` | Minimal | Community maintains |
| Popular library, no types | Write `.d.ts`, contribute to DefinitelyTyped | Medium | You maintain initially, then community |
| Internal legacy code | Write `.d.ts` alongside code | Low | You maintain |
| Prototype/throwaway code | Use `any` or `// @ts-ignore` | None | Not recommended for production |

### Best Practices for Writing Declaration Files

**1. Start minimal, expand as needed**

Don't try to type the entire library at once. Type only what you use:

```typescript
// Start with this:
declare module 'awesome-charts' {
  export function LineChart(props: any): any;
}

// Expand as you use more features:
declare module 'awesome-charts' {
  interface LineChartProps {
    data: any[];
    xKey: string;
    yKey: string;
  }
  export function LineChart(props: LineChartProps): React.ReactElement;
}
```

**2. Use generics for flexible types**

```typescript
// Good: Generic type parameter
interface ChartProps<T> {
  data: T[];
  xKey: keyof T;
  yKey: keyof T;
}

// Bad: Hardcoded type
interface ChartProps {
  data: any[];
  xKey: string;
  yKey: string;
}
```

**3. Document with JSDoc comments**

```typescript
/**
 * Calculate percentage change between two values.
 * 
 * @param current - Current value
 * @param previous - Previous value
 * @returns Percentage change (e.g., 25 for 25% increase)
 * @throws {Error} If previous is negative
 * 
 * @example
 * ```ts
 * calculateChange(125, 100)  // Returns 25
 * calculateChange(75, 100)   // Returns -25
 * ```
 */
export function calculateChange(current: number, previous: number): number;
```

**4. Use `unknown` instead of `any` when possible**

```typescript
// Good: Forces type checking
function processData(data: unknown) {
  if (typeof data === 'object' && data !== null) {
    // Now TypeScript knows data is an object
  }
}

// Bad: Bypasses type checking
function processData(data: any) {
  // TypeScript allows anything
}
```

**5. Contribute back to DefinitelyTyped**

If you write types for a popular library, contribute them:

```bash
# Fork DefinitelyTyped repo
git clone https://github.com/DefinitelyTyped/DefinitelyTyped.git

# Create types
cd DefinitelyTyped/types
mkdir awesome-charts
cd awesome-charts

# Copy your .d.ts file
cp ~/my-project/src/types/awesome-charts.d.ts index.d.ts

# Add package.json and tsconfig.json
# Submit pull request
```

### The Third-Party Types Journey

| Scenario | Before | After | Effort |
|----------|--------|-------|--------|
| Built-in types | ✅ Works | ✅ Works | None |
| DefinitelyTyped | ❌ No types | ✅ Full types | `npm install` |
| No types available | ❌ `any` everywhere | ✅ Custom types | Write `.d.ts` |
| Legacy internal code | ❌ `any` everywhere | ✅ Typed | Write `.d.ts` |

### Lessons Learned: Handling Third-Party Types

**1. Check for types first**: Always try `npm i @types/package-name` before writing your own

**2. Start minimal**: Type only what you use, expand as needed

**3. Use generics**: Make your types flexible and reusable

**4. Document thoroughly**: JSDoc comments help future developers (including yourself)

**5. Contribute back**: If you write types for a popular library, share them with the community

**6. Don't use `any` as a permanent solution**: It defeats the purpose of TypeScript

**7. Prefer `unknown` over `any`**: Forces you to validate types before use

**Next**: In Section 9.4, we'll configure `tsconfig.json` for optimal type checking and developer experience.

## tsconfig settings that matter

## tsconfig Settings That Matter

You've migrated your code to TypeScript, added types for third-party libraries, and everything compiles. But TypeScript has **dozens** of compiler options, and the defaults are often too permissive for production code.

This section focuses on the **20% of settings that give you 80% of the value**—the options that catch real bugs and improve developer experience.

### The Problem with Default Settings

**The Failure: TypeScript Allows Unsafe Code**

Here's code that compiles with default TypeScript settings but crashes at runtime:

```typescript
// This compiles with default tsconfig.json
function getUserName(user: any) {
  return user.name.toUpperCase();
}

const user = null;
const name = getUserName(user);  // ← Runtime crash
console.log(name);
```

**Browser Console Output**:
```
TypeError: Cannot read properties of null (reading 'name')
  at getUserName (index.ts:2)
  at index.ts:6
```

**Diagnostic Analysis: Why TypeScript Didn't Catch This**

**Let's parse this evidence**:

1. **What the developer expected**: TypeScript to catch the `null` access

2. **What TypeScript allowed**: `any` type bypasses all type checking

3. **Root cause identified**: Default `tsconfig.json` is too permissive

4. **Why default settings fail**: They prioritize ease of migration over safety

5. **What we need**: Stricter compiler options that catch these bugs

### Iteration 1: The Essential Strict Settings

**Current state recap**: Default TypeScript settings allow unsafe code to compile.

**Goal**: Enable strict type checking to catch bugs at compile time.

**The Minimal Production tsconfig.json**:

```json
{
  "compilerOptions": {
    /* Language and Environment */
    "target": "ES2020",                    // Modern JavaScript features
    "lib": ["ES2020", "DOM", "DOM.Iterable"],  // Available APIs
    "jsx": "react-jsx",                    // React 17+ JSX transform

    /* Modules */
    "module": "ESNext",                    // Use ES modules
    "moduleResolution": "bundler",         // For Vite/modern bundlers
    "resolveJsonModule": true,             // Import JSON files
    "allowImportingTsExtensions": true,    // Import .ts/.tsx files

    /* Type Checking - THE MOST IMPORTANT SECTION */
    "strict": true,                        // Enable all strict checks
    "noUncheckedIndexedAccess": true,      // Array access returns T | undefined
    "noImplicitOverride": true,            // Require 'override' keyword
    "noPropertyAccessFromIndexSignature": true,  // Prevent typos in property access

    /* Emit */
    "noEmit": true,                        // Let Vite handle compilation
    "sourceMap": true,                     // Generate source maps for debugging

    /* Interop Constraints */
    "esModuleInterop": true,               // Better CommonJS interop
    "allowSyntheticDefaultImports": true,  // Allow default imports from modules
    "forceConsistentCasingInFileNames": true,  // Prevent case-sensitivity issues

    /* Skip Lib Check */
    "skipLibCheck": true                   // Skip type checking of .d.ts files
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

**What `"strict": true` Actually Enables**

The `strict` flag is a shorthand that enables these individual checks:

1. **`strictNullChecks`**: `null` and `undefined` are not assignable to other types
2. **`strictFunctionTypes`**: Function parameters are checked contravariantly
3. **`strictBindCallApply`**: `bind`, `call`, `apply` are type-checked
4. **`strictPropertyInitialization`**: Class properties must be initialized
5. **`noImplicitAny`**: Variables must have explicit or inferred types
6. **`noImplicitThis`**: `this` must have an explicit type
7. **`alwaysStrict`**: Emit `"use strict"` in output
8. **`useUnknownInCatchVariables`**: Catch variables are `unknown` instead of `any`

Let's see each in action.

### Strict Check 1: strictNullChecks

**Before (Disabled)**:

```typescript
// Compiles without strictNullChecks
function getUserName(user: { name: string } | null) {
  return user.name.toUpperCase();  // ← No error, crashes at runtime
}
```

**After (Enabled)**:

```typescript
// Error with strictNullChecks
function getUserName(user: { name: string } | null) {
  return user.name.toUpperCase();  // ← Error: Object is possibly 'null'
}

// Fixed version
function getUserName(user: { name: string } | null) {
  if (user === null) {
    return 'Anonymous';
  }
  return user.name.toUpperCase();  // ← Now safe
}
```

**Terminal Output**:
```bash
src/utils.ts:2:10 - error TS2531:
Object is possibly 'null'.

2   return user.name.toUpperCase();
           ~~~~
```

### Strict Check 2: noImplicitAny

**Before (Disabled)**:

```typescript
// Compiles without noImplicitAny
function calculateTotal(items) {  // ← items is implicitly 'any'
  return items.reduce((sum, item) => sum + item.price, 0);
}
```

**After (Enabled)**:

```typescript
// Error with noImplicitAny
function calculateTotal(items) {  // ← Error: Parameter 'items' implicitly has an 'any' type
  return items.reduce((sum, item) => sum + item.price, 0);
}

// Fixed version
interface Item {
  price: number;
}

function calculateTotal(items: Item[]) {
  return items.reduce((sum, item) => sum + item.price, 0);
}
```

**Terminal Output**:
```bash
src/utils.ts:1:24 - error TS7006:
Parameter 'items' implicitly has an 'any' type.

1 function calculateTotal(items) {
                          ~~~~~
```

### Strict Check 3: strictPropertyInitialization

**Before (Disabled)**:

```typescript
// Compiles without strictPropertyInitialization
class UserProfile {
  name: string;  // ← Not initialized, will be undefined
  email: string;

  constructor(name: string) {
    this.name = name;
    // Forgot to initialize email
  }

  sendEmail() {
    return this.email.toLowerCase();  // ← Crashes: undefined.toLowerCase()
  }
}
```

**After (Enabled)**:

```typescript
// Error with strictPropertyInitialization
class UserProfile {
  name: string;
  email: string;  // ← Error: Property 'email' has no initializer

  constructor(name: string) {
    this.name = name;
  }
}

// Fixed version 1: Initialize in constructor
class UserProfile {
  name: string;
  email: string;

  constructor(name: string, email: string) {
    this.name = name;
    this.email = email;
  }
}

// Fixed version 2: Provide default value
class UserProfile {
  name: string;
  email: string = '';  // ← Default value

  constructor(name: string) {
    this.name = name;
  }
}

// Fixed version 3: Mark as optional
class UserProfile {
  name: string;
  email?: string;  // ← Optional property

  constructor(name: string) {
    this.name = name;
  }

  sendEmail() {
    if (this.email) {  // ← Must check before use
      return this.email.toLowerCase();
    }
    throw new Error('Email not set');
  }
}
```

**Terminal Output**:
```bash
src/models/UserProfile.ts:3:3 - error TS2564:
Property 'email' has no initializer and is not definitely assigned in the constructor.

3   email: string;
    ~~~~~
```

### Iteration 2: Additional Safety Checks

**Current state recap**: We've enabled `strict: true`, which catches most common bugs.

**Current limitation**: There are additional checks that aren't included in `strict` but are valuable for production code.

**Additional Recommended Settings**:

### Check 4: noUncheckedIndexedAccess

**The Problem**: Array access can return `undefined`, but TypeScript doesn't enforce checking.

**Before (Disabled)**:

```typescript
// Compiles without noUncheckedIndexedAccess
const users = ['Alice', 'Bob'];
const thirdUser = users[2];  // ← undefined, but TypeScript thinks it's string
console.log(thirdUser.toUpperCase());  // ← Crashes
```

**After (Enabled)**:

```typescript
// Error with noUncheckedIndexedAccess
const users = ['Alice', 'Bob'];
const thirdUser = users[2];  // ← Type is string | undefined
console.log(thirdUser.toUpperCase());  // ← Error: Object is possibly 'undefined'

// Fixed version
const users = ['Alice', 'Bob'];
const thirdUser = users[2];
if (thirdUser !== undefined) {
  console.log(thirdUser.toUpperCase());
}

// Or use optional chaining
console.log(users[2]?.toUpperCase());
```

**Terminal Output**:
```bash
src/utils.ts:3:13 - error TS2532:
Object is possibly 'undefined'.

3 console.log(thirdUser.toUpperCase());
              ~~~~~~~~~~
```

### Check 5: noPropertyAccessFromIndexSignature

**The Problem**: Typos in property names go undetected.

**Before (Disabled)**:

```typescript
// Compiles without noPropertyAccessFromIndexSignature
interface Config {
  apiUrl: string;
  timeout: number;
  [key: string]: any;  // ← Index signature allows any property
}

const config: Config = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
};

console.log(config.apiUrl);   // ← OK
console.log(config.apiUrll);  // ← Typo, but compiles (returns undefined)
```

**After (Enabled)**:

```typescript
// Error with noPropertyAccessFromIndexSignature
interface Config {
  apiUrl: string;
  timeout: number;
  [key: string]: any;
}

const config: Config = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
};

console.log(config.apiUrl);   // ← OK (known property)
console.log(config.apiUrll);  // ← Error: Property 'apiUrll' does not exist
console.log(config['apiUrll']);  // ← OK (explicit index access)
```

**Terminal Output**:
```bash
src/config.ts:11:20 - error TS2339:
Property 'apiUrll' does not exist on type 'Config'.
Did you mean 'apiUrl'?

11 console.log(config.apiUrll);
                      ~~~~~~~~
```

### Check 6: noImplicitOverride

**The Problem**: Accidentally overriding parent class methods without realizing it.

**Before (Disabled)**:

```typescript
// Compiles without noImplicitOverride
class BaseComponent {
  render() {
    return '<div>Base</div>';
  }
}

class CustomComponent extends BaseComponent {
  // Typo: meant to override render(), but wrote rendor()
  rendor() {  // ← New method, not an override
    return '<div>Custom</div>';
  }
}

const component = new CustomComponent();
console.log(component.render());  // ← Still calls base class method
```

**After (Enabled)**:

```typescript
// Error with noImplicitOverride
class BaseComponent {
  render() {
    return '<div>Base</div>';
  }
}

class CustomComponent extends BaseComponent {
  override render() {  // ← Must use 'override' keyword
    return '<div>Custom</div>';
  }
}

// If you typo the method name:
class CustomComponent extends BaseComponent {
  override rendor() {  // ← Error: Method 'rendor' does not override any base class method
    return '<div>Custom</div>';
  }
}
```

**Terminal Output**:
```bash
src/components/CustomComponent.ts:8:12 - error TS4113:
This member cannot have an 'override' modifier because it is not declared in the base class 'BaseComponent'.

8   override rendor() {
             ~~~~~~
```

### Iteration 3: Performance and Developer Experience Settings

**Current state recap**: We have strict type checking enabled, catching most bugs at compile time.

**Goal**: Optimize TypeScript for faster compilation and better editor experience.

**Performance Settings**:

```json
{
  "compilerOptions": {
    /* ... previous settings ... */

    /* Performance */
    "skipLibCheck": true,              // Skip type checking of .d.ts files (huge speedup)
    "incremental": true,               // Enable incremental compilation
    "tsBuildInfoFile": ".tsbuildinfo", // Cache file for incremental builds

    /* Editor Experience */
    "noErrorTruncation": true,         // Show full error messages
    "pretty": true,                    // Colorize error messages
    
    /* Path Mapping (for cleaner imports) */
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"]
    }
  }
}
```

**Using Path Mapping**:

**Before**:

```typescript
// Ugly relative imports
import { ProductCard } from '../../../components/ProductCard';
import { formatPrice } from '../../../utils/formatPrice';
import { Product } from '../../../types/product';
```

**After**:

```typescript
// Clean absolute imports
import { ProductCard } from '@components/ProductCard';
import { formatPrice } from '@utils/formatPrice';
import { Product } from '@types/product';
```

**Note**: You also need to configure your bundler (Vite, Webpack) to resolve these paths:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@utils': path.resolve(__dirname, './src/utils'),
      '@types': path.resolve(__dirname, './src/types'),
    },
  },
});
```

### The Complete Production tsconfig.json

Here's the final, production-ready configuration:

```json
{
  "compilerOptions": {
    /* Language and Environment */
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",

    /* Modules */
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "allowImportingTsExtensions": true,

    /* Type Checking - Maximum Safety */
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,

    /* Emit */
    "noEmit": true,
    "sourceMap": true,

    /* Interop Constraints */
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,

    /* Performance */
    "skipLibCheck": true,
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo",

    /* Editor Experience */
    "noErrorTruncation": true,
    "pretty": true,

    /* Path Mapping */
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"],
      "@hooks/*": ["src/hooks/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist", "build"]
}
```

### Settings to Avoid (Common Mistakes)

**1. Don't disable strict checks in production**

```json
// ❌ Bad: Defeats the purpose of TypeScript
{
  "compilerOptions": {
    "strict": false,
    "noImplicitAny": false,
    "strictNullChecks": false
  }
}
```

**2. Don't use `any` as an escape hatch**

```typescript
// ❌ Bad: Bypasses type checking
function processData(data: any) {
  return data.value.toUpperCase();
}

// ✅ Good: Use unknown and validate
function processData(data: unknown) {
  if (typeof data === 'object' && data !== null && 'value' in data) {
    const obj = data as { value: unknown };
    if (typeof obj.value === 'string') {
      return obj.value.toUpperCase();
    }
  }
  throw new Error('Invalid data');
}
```

**3. Don't ignore errors with @ts-ignore**

```typescript
// ❌ Bad: Hides the problem
// @ts-ignore
const result = user.name.toUpperCase();

// ✅ Good: Fix the problem
const result = user?.name?.toUpperCase() ?? 'Anonymous';
```

**4. Don't set target too low**

```json
// ❌ Bad: Generates bloated code
{
  "compilerOptions": {
    "target": "ES5"  // Transpiles async/await to generators
  }
}

// ✅ Good: Modern target for modern browsers
{
  "compilerOptions": {
    "target": "ES2020"  // Native async/await, optional chaining, etc.
  }
}
```

### When to Relax Strict Settings

**Scenario 1: Migrating Large Codebase**

Start with loose settings, gradually tighten:

```json
// Phase 1: Basic TypeScript
{
  "compilerOptions": {
    "strict": false,
    "noImplicitAny": true  // Start with just this
  }
}

// Phase 2: Add null checks
{
  "compilerOptions": {
    "strict": false,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}

// Phase 3: Full strict mode
{
  "compilerOptions": {
    "strict": true
  }
}
```

**Scenario 2: Third-Party Library Issues**

Use `skipLibCheck` to ignore type errors in dependencies:

```json
{
  "compilerOptions": {
    "strict": true,
    "skipLibCheck": true  // Ignore errors in node_modules
  }
}
```

**Scenario 3: Prototyping**

For throwaway code, you can be more permissive:

```json
{
  "compilerOptions": {
    "strict": false,
    "noImplicitAny": false
  }
}
```

But **never** deploy this to production.

### The tsconfig Journey

| Phase | Settings | Bugs Caught | Compile Time | Developer Experience |
|-------|----------|-------------|--------------|---------------------|
| Default | Minimal | Few | Fast | Poor (no autocomplete) |
| Basic | `noImplicitAny` | Some | Fast | Better |
| Strict | `strict: true` | Most | Medium | Good |
| Production | All checks | Maximum | Medium | Excellent |

### Verification: Measuring the Impact

**Before (Loose Settings)**:

```bash
# Compile time
tsc --noEmit: 2.3s

# Bugs caught at compile time: 12
# Bugs found in production: 8
```

**After (Strict Settings)**:

```bash
# Compile time
tsc --noEmit: 2.8s (20% slower, but worth it)

# Bugs caught at compile time: 20
# Bugs found in production: 0
```

**Expected vs. Actual Improvement**:

**Before (Default Settings)**:
- Fast compilation
- Many runtime errors
- Poor editor experience
- Frequent production bugs

**After (Strict Settings)**:
- Slightly slower compilation (20%)
- Most bugs caught at compile time
- Excellent editor experience (autocomplete, refactoring)
- Rare production bugs

### Lessons Learned: tsconfig Settings

**1. Start strict**: Enable `strict: true` from day one on new projects

**2. Add extra checks**: `noUncheckedIndexedAccess`, `noImplicitOverride`, `noPropertyAccessFromIndexSignature`

**3. Use path mapping**: Clean imports improve readability and refactoring

**4. Skip lib check**: `skipLibCheck: true` dramatically speeds up compilation

**5. Never use `any`**: Use `unknown` and validate instead

**6. Don't ignore errors**: Fix them or use proper type guards

**7. Measure the impact**: Track bugs caught at compile time vs. runtime

**8. Migrate gradually**: For large codebases, tighten settings incrementally

**9. Document exceptions**: If you must relax a setting, document why

**10. Review regularly**: As TypeScript evolves, new checks become available

### The Complete TypeScript Journey (Chapters 8-9)

| Chapter | Focus | Key Takeaway |
|---------|-------|--------------|
| 8 | TypeScript Essentials | Types prevent runtime errors |
| 9.1 | Migrating JavaScript | Incremental migration strategy |
| 9.2 | Type-Safe APIs | Zod for runtime validation |
| 9.3 | Third-Party Types | Write `.d.ts` when needed |
| 9.4 | tsconfig Settings | Strict settings catch bugs |

### Final Implementation: Production-Ready TypeScript Setup

**Project Structure**:
```
my-app/
├── src/
│   ├── components/
│   │   ├── ProductCard.tsx
│   │   ├── ProductList.tsx
│   │   └── AnalyticsDashboard.tsx
│   ├── utils/
│   │   ├── api.ts
│   │   └── formatPrice.ts
│   ├── types/
│   │   ├── product.ts
│   │   ├── awesome-charts.d.ts
│   │   └── analytics.d.ts
│   └── hooks/
│       └── useProducts.ts
├── tsconfig.json              ← Strict settings
├── vite.config.ts             ← Path aliases
└── package.json
```

**tsconfig.json** (Final):
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "allowImportingTsExtensions": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "noEmit": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo",
    "noErrorTruncation": true,
    "pretty": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"],
      "@hooks/*": ["src/hooks/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist", "build"]
}
```

**Decision Framework: TypeScript Configuration**

| Scenario | Recommended Settings | Rationale |
|----------|---------------------|-----------|
| New project | Full strict mode | Catch bugs early, best DX |
| Large migration | Gradual tightening | Avoid overwhelming team |
| Prototype | Loose settings | Speed over safety |
| Production app | Full strict + extras | Maximum safety |
| Library | Strict + declaration emit | Type safety for consumers |

### What We've Accomplished

**Chapter 9 Journey**:

1. **Section 9.1**: Migrated JavaScript to TypeScript incrementally
2. **Section 9.2**: Built type-safe API clients with Zod
3. **Section 9.3**: Added types for third-party libraries
4. **Section 9.4**: Configured TypeScript for maximum safety

**From Zero to Hero**:
- Started with untyped JavaScript
- Added basic TypeScript types
- Implemented runtime validation
- Handled third-party libraries
- Configured strict type checking
- Built a production-ready TypeScript setup

**The Result**: A fully type-safe React application that catches bugs at compile time, provides excellent developer experience, and rarely crashes in production.

**Next Steps**: In Part III (Chapters 11-13), we'll tackle state management—local state, global state with Zustand, and server state with React Query. TypeScript will make all of these patterns safer and more maintainable.
