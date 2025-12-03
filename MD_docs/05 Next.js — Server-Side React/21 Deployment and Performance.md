# Chapter 21: Deployment and Performance

## Deploying to Vercel (the path of least resistance)

## Phase 1: Establish the Reference Implementation

You've built a Next.js application. It works perfectly on `localhost:3000`. Now you need to deploy it to production so real users can access it. This is where many developers encounter their first production failure: **the build that works locally fails in production**.

We'll use a realistic e-commerce product catalog as our reference implementation—the same one we've been building through Chapters 16-20. It has:

- Server Components fetching product data
- Client Components for interactive cart
- API routes for checkout
- Authentication with NextAuth.js
- Image optimization
- Environment variables for API keys

Let's see what happens when we try to deploy this to production for the first time.

### The Reference Implementation: Product Catalog

Our application structure:

**Project Structure**:
```
product-catalog/
├── app/
│   ├── products/
│   │   ├── page.tsx          ← Server Component (product list)
│   │   └── [id]/
│   │       └── page.tsx      ← Server Component (product detail)
│   ├── cart/
│   │   └── page.tsx          ← Client Component (shopping cart)
│   ├── api/
│   │   ├── checkout/
│   │   │   └── route.ts      ← API route
│   │   └── auth/
│   │       └── [...nextauth]/
│   │           └── route.ts  ← NextAuth.js
│   └── layout.tsx
├── components/
│   ├── ProductCard.tsx       ← Client Component
│   └── AddToCartButton.tsx   ← Client Component
├── lib/
│   ├── db.ts                 ← Database client
│   └── stripe.ts             ← Stripe client
├── public/
│   └── images/               ← Product images
├── .env.local                ← Local environment variables
├── next.config.js
└── package.json
```

Here's our product listing page that works perfectly locally:

```tsx
// app/products/page.tsx
import { db } from '@/lib/db';
import ProductCard from '@/components/ProductCard';

export default async function ProductsPage() {
  // Server Component - fetch directly
  const products = await db.product.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' },
  });

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {products.map((product) => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  );
}
```

```tsx
// components/ProductCard.tsx
'use client';

import Image from 'next/image';
import { useCart } from '@/hooks/useCart';

interface ProductCardProps {
  product: {
    id: string;
    name: string;
    price: number;
    imageUrl: string;
  };
}

export default function ProductCard({ product }: ProductCardProps) {
  const { addItem } = useCart();

  return (
    <div className="border rounded-lg p-4">
      <Image
        src={product.imageUrl}
        alt={product.name}
        width={300}
        height={300}
        className="w-full h-48 object-cover rounded"
      />
      <h3 className="text-xl font-semibold mt-4">{product.name}</h3>
      <p className="text-gray-600">${product.price}</p>
      <button
        onClick={() => addItem(product)}
        className="mt-4 w-full bg-blue-600 text-white py-2 rounded hover:bg-blue-700"
      >
        Add to Cart
      </button>
    </div>
  );
}
```

```typescript
// lib/db.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = global as unknown as { prisma: PrismaClient };

export const db = globalForPrisma.prisma || new PrismaClient();

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db;
```

Our `.env.local` file contains all our secrets:

```bash
# .env.local
DATABASE_URL="postgresql://user:password@localhost:5432/products"
NEXTAUTH_SECRET="super-secret-key-for-local-dev"
NEXTAUTH_URL="http://localhost:3000"
STRIPE_SECRET_KEY="sk_test_..."
STRIPE_PUBLISHABLE_KEY="pk_test_..."
```

Locally, everything works:
- Products load instantly
- Images display correctly
- Cart functionality works
- Checkout processes payments

Now let's deploy to production.

## The First Deployment: When Everything Breaks

## Phase 2: Diagnostic Analysis - The Production Build Failure

### Iteration 0: The Naive First Deployment

You push your code to GitHub, connect it to Vercel, and click "Deploy". Here's what happens:

**Terminal Output (Vercel Build Log)**:
```bash
[12:34:56.789] Running build in Washington, D.C., USA (East) – iad1
[12:34:57.123] Cloning github.com/yourname/product-catalog (Branch: main, Commit: abc123)
[12:34:58.456] Installing dependencies...
[12:35:12.789] Dependencies installed
[12:35:13.123] Running "npm run build"
[12:35:15.456] > product-catalog@0.1.0 build
[12:35:15.456] > next build
[12:35:16.789] 
[12:35:16.789]    ▲ Next.js 14.0.0
[12:35:16.789] 
[12:35:18.123]    Creating an optimized production build ...
[12:35:25.456] ✓ Compiled successfully
[12:35:26.789] 
[12:35:26.789]    Linting and checking validity of types ...
[12:35:28.123] 
[12:35:28.123] Failed to compile.
[12:35:28.123] 
[12:35:28.123] ./app/products/page.tsx:8:23
[12:35:28.123] Type error: Property 'product' does not exist on type 'PrismaClient'
[12:35:28.123] 
[12:35:28.123]   6 | export default async function ProductsPage() {
[12:35:28.123]   7 |   const products = await db.product.findMany({
[12:35:28.123] > 8 |     where: { published: true },
[12:35:28.123]     |                       ^
[12:35:28.123]   9 |     orderBy: { createdAt: 'desc' },
[12:35:28.123]  10 |   });
[12:35:28.123]  11 | 
[12:35:28.123] 
[12:35:28.456] Error: Command "npm run build" exited with 1
[12:35:28.789] BUILD FAILED
```

### Diagnostic Analysis: Reading the Build Failure

**Build Behavior**:
The build process starts successfully, installs dependencies, begins compilation, but fails during type checking.

**Terminal Evidence**:
- Build reaches "Linting and checking validity of types"
- TypeScript error: `Property 'product' does not exist on type 'PrismaClient'`
- Error location: `app/products/page.tsx:8:23`
- Build exits with code 1 (failure)

**What Happened Locally vs. Production**:
- **Locally**: Prisma Client was generated with `npx prisma generate`, types exist
- **Production**: Prisma Client was never generated, types don't exist

**Let's parse this evidence**:

1. **What the build process experiences**: TypeScript can't find the Prisma types because the Prisma Client hasn't been generated yet.

2. **What the error reveals**: The build process doesn't know about your database schema. Prisma generates TypeScript types from your schema, but that generation step never ran.

3. **Root cause identified**: Missing build step—Prisma Client generation must happen before Next.js build.

4. **Why the current approach can't solve this**: Simply pushing code isn't enough. Production builds need explicit instructions for multi-step build processes.

5. **What we need**: A `postinstall` script to generate Prisma Client after dependencies are installed but before Next.js builds.

### Iteration 1: Adding the Prisma Generation Step

**Before** (package.json):

```json
{
  "name": "product-catalog",
  "version": "0.1.0",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "@prisma/client": "^5.7.0",
    "next": "14.0.0",
    "react": "^18.2.0"
  },
  "devDependencies": {
    "prisma": "^5.7.0",
    "typescript": "^5.3.0"
  }
}
```

**Problem**: No instruction to generate Prisma Client during build.

**After** (package.json):

```json
{
  "name": "product-catalog",
  "version": "0.1.0",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "postinstall": "prisma generate"
  },
  "dependencies": {
    "@prisma/client": "^5.7.0",
    "next": "14.0.0",
    "react": "^18.2.0"
  },
  "devDependencies": {
    "prisma": "^5.7.0",
    "typescript": "^5.3.0"
  }
}
```

**Improvement**: `postinstall` script runs automatically after `npm install`, generating Prisma Client before the build.

Push this change and redeploy:

**Terminal Output (Second Deployment)**:
```bash
[12:45:12.123] Installing dependencies...
[12:45:25.456] 
[12:45:25.456] > product-catalog@0.1.0 postinstall
[12:45:25.456] > prisma generate
[12:45:26.789] 
[12:45:26.789] Prisma schema loaded from prisma/schema.prisma
[12:45:28.123] ✔ Generated Prisma Client (5.7.0) to ./node_modules/@prisma/client
[12:45:28.456] 
[12:45:28.789] Dependencies installed
[12:45:29.123] Running "npm run build"
[12:45:35.456] ✓ Compiled successfully
[12:45:36.789] ✓ Linting and checking validity of types
[12:45:38.123] ✓ Collecting page data
[12:45:42.456] ✓ Generating static pages (5/5)
[12:45:43.789] ✓ Collecting build traces
[12:45:44.123] ✓ Finalizing page optimization
[12:45:44.456] 
[12:45:44.456] Route (app)                              Size     First Load JS
[12:45:44.456] ┌ ○ /                                    142 B          87.2 kB
[12:45:44.456] ├ ○ /products                            1.2 kB         88.3 kB
[12:45:44.456] └ ○ /products/[id]                       890 B          87.9 kB
[12:45:44.456] 
[12:45:44.456] ○  (Static)  prerendered as static content
[12:45:44.456] 
[12:45:44.789] BUILD SUCCESSFUL
[12:45:45.123] Deploying...
[12:45:48.456] Deployment ready: https://product-catalog-abc123.vercel.app
```

**Success!** The build completes. But when you visit the deployed URL...

**Browser Behavior**:
- Page loads with layout and header
- Product grid shows loading state
- After 10 seconds: "500 Internal Server Error"

**Browser Console**:
```
Failed to load resource: the server responded with a status of 500 ()
GET https://product-catalog-abc123.vercel.app/products 500
```

**Vercel Function Logs** (in Vercel Dashboard):
```
[ERROR] PrismaClientInitializationError: 
Can't reach database server at `localhost:5432`

Please make sure your database server is running at `localhost:5432`.

  at PrismaClient.connect (/var/task/.next/server/chunks/123.js:45678:23)
  at async ProductsPage (/var/task/.next/server/app/products/page.js:12:34)
```

### Diagnostic Analysis: Reading the Runtime Failure

**Browser Behavior**:
Page structure loads (layout, navigation), but product data never appears. After timeout, shows 500 error.

**Browser Console Output**:
HTTP 500 status from `/products` route—server-side error, not client-side.

**Vercel Function Logs Evidence**:
- Error type: `PrismaClientInitializationError`
- Specific issue: "Can't reach database server at `localhost:5432`"
- Error location: During Prisma connection attempt in Server Component

**Let's parse this evidence**:

1. **What the user experiences**: Page loads partially, then fails. No products display.

2. **What the console reveals**: 500 error means server-side failure. The problem isn't in the browser—it's in the serverless function.

3. **What the logs show**: Prisma is trying to connect to `localhost:5432`. In production, there is no localhost database. The production environment doesn't have access to your local PostgreSQL.

4. **Root cause identified**: Environment variables from `.env.local` don't exist in production. The `DATABASE_URL` is still pointing to localhost.

5. **Why the current approach can't solve this**: `.env.local` files are never committed to Git (and shouldn't be—they contain secrets). Production needs its own environment variables.

6. **What we need**: Configure environment variables in Vercel's dashboard, pointing to a production database.

### Iteration 2: Configuring Production Environment Variables

First, we need a production database. For this example, we'll use a hosted PostgreSQL database (Vercel Postgres, Supabase, Railway, or any provider).

**Step 1: Create Production Database**

Let's say you've created a Vercel Postgres database. Vercel provides these credentials:

```bash
# Production database credentials (from Vercel Postgres)
POSTGRES_URL="postgres://default:abc123@ep-cool-name-123456.us-east-1.postgres.vercel-storage.com:5432/verceldb"
POSTGRES_PRISMA_URL="postgres://default:abc123@ep-cool-name-123456.us-east-1.postgres.vercel-storage.com:5432/verceldb?pgbouncer=true&connect_timeout=15"
POSTGRES_URL_NON_POOLING="postgres://default:abc123@ep-cool-name-123456.us-east-1.postgres.vercel-storage.com:5432/verceldb"
```

**Step 2: Add Environment Variables in Vercel Dashboard**

Navigate to your project in Vercel:
1. Go to Project Settings → Environment Variables
2. Add each variable:

| Key | Value | Environment |
|-----|-------|-------------|
| `DATABASE_URL` | `postgres://default:abc123@ep-cool-name...` | Production, Preview, Development |
| `NEXTAUTH_SECRET` | `[generate new secret for production]` | Production |
| `NEXTAUTH_URL` | `https://product-catalog-abc123.vercel.app` | Production |
| `STRIPE_SECRET_KEY` | `sk_live_...` | Production |
| `STRIPE_PUBLISHABLE_KEY` | `pk_live_...` | Production, Preview, Development |

**Critical**: Use different secrets for production vs. development. Never use test API keys in production.

**Step 3: Run Database Migrations**

Your production database is empty. You need to apply your Prisma schema:

```bash
# Set production database URL temporarily
export DATABASE_URL="postgres://default:abc123@ep-cool-name..."

# Run migrations against production database
npx prisma migrate deploy

# Seed initial data (if you have a seed script)
npx prisma db seed
```

**Step 4: Redeploy**

Vercel automatically redeploys when you add environment variables. Or trigger manually:

```bash
# Trigger redeployment
git commit --allow-empty -m "Trigger redeploy with env vars"
git push
```

**Terminal Output (Third Deployment)**:
```bash
[13:15:44.456] BUILD SUCCESSFUL
[13:15:45.123] Deploying...
[13:15:48.456] Deployment ready: https://product-catalog-abc123.vercel.app
```

Now visit the deployed URL:

**Browser Behavior**:
- Page loads
- Products appear!
- Images... are broken (404 errors)
- Cart button works
- Clicking "Add to Cart" → 500 error

**Browser Console**:
```
GET https://product-catalog-abc123.vercel.app/images/product-1.jpg 404 (Not Found)
GET https://product-catalog-abc123.vercel.app/images/product-2.jpg 404 (Not Found)
GET https://product-catalog-abc123.vercel.app/images/product-3.jpg 404 (Not Found)

POST https://product-catalog-abc123.vercel.app/api/cart/add 500 (Internal Server Error)
```

**Vercel Function Logs**:
```
[ERROR] Error: NEXTAUTH_SECRET is not set
  at /var/task/.next/server/chunks/456.js:12345:67
  at async POST (/var/task/.next/server/app/api/cart/add/route.js:23:45)
```

### Diagnostic Analysis: Multiple Remaining Issues

We have two separate problems:

**Problem 1: Missing Images**
- **Browser behavior**: 404 errors for all product images
- **Console evidence**: Requests to `/images/product-X.jpg` return 404
- **Root cause**: Images in `public/images/` weren't committed to Git (likely in `.gitignore`), or they're stored locally and not in the repository

**Problem 2: API Route Failure**
- **Browser behavior**: Cart operations fail with 500 error
- **Function logs**: `NEXTAUTH_SECRET is not set`
- **Root cause**: The API route uses NextAuth.js session verification, but we only set `NEXTAUTH_SECRET` for "Production" environment, not "Preview"

Let's fix both issues.

### Iteration 3: Fixing Images and Environment Variable Scope

**Fix 1: Ensure Images Are Committed**

Check your `.gitignore`:

```bash
# .gitignore
node_modules/
.next/
.env*.local
.vercel

# Make sure this line is NOT present:
# public/images/
```

If images were ignored, remove that line and commit them:

```bash
git add public/images/
git commit -m "Add product images"
git push
```

**Fix 2: Set Environment Variables for All Environments**

In Vercel Dashboard, edit each environment variable to include all environments:
- Production ✓
- Preview ✓
- Development ✓

This ensures preview deployments (from pull requests) also have access to necessary secrets.

**Alternative**: Use different values per environment:

```bash
# Production
NEXTAUTH_SECRET="production-secret-very-long-and-random"
NEXTAUTH_URL="https://product-catalog.vercel.app"

# Preview
NEXTAUTH_SECRET="preview-secret-different-from-prod"
NEXTAUTH_URL="https://product-catalog-git-main-yourname.vercel.app"

# Development
NEXTAUTH_SECRET="development-secret-for-local"
NEXTAUTH_URL="http://localhost:3000"
```

**Terminal Output (Fourth Deployment)**:
```bash
[13:45:44.456] BUILD SUCCESSFUL
[13:45:45.123] Deploying...
[13:45:48.456] Deployment ready: https://product-catalog-abc123.vercel.app
```

Visit the deployed URL again:

**Browser Behavior**:
- Page loads ✓
- Products appear ✓
- Images display ✓
- Cart button works ✓
- Add to cart succeeds ✓
- Checkout... takes 30 seconds and times out

**Browser Console**:
```
POST https://product-catalog-abc123.vercel.app/api/checkout 504 (Gateway Timeout)
```

**Vercel Function Logs**:
```
[ERROR] Task timed out after 10.00 seconds
  at async POST (/var/task/.next/server/app/api/checkout/route.js:34:56)
```

### Diagnostic Analysis: Serverless Function Timeout

**Browser Behavior**:
Checkout request hangs, eventually times out with 504 Gateway Timeout.

**Function Logs Evidence**:
- Error: "Task timed out after 10.00 seconds"
- Location: Checkout API route

**What's Happening**:
Vercel's free tier has a 10-second timeout for serverless functions. Your checkout process (which might involve Stripe payment processing, database writes, email sending) takes longer than 10 seconds.

**Root cause identified**: Serverless functions have execution time limits. Long-running operations need to be optimized or moved to background jobs.

**Solutions**:
1. **Optimize the checkout flow**: Make it faster (parallel operations, reduce database queries)
2. **Upgrade Vercel plan**: Pro plan has 60-second timeout, Enterprise has 900 seconds
3. **Use background jobs**: Offload non-critical operations (email sending) to a queue
4. **Stream responses**: Use streaming to keep connection alive while processing

For now, let's optimize the checkout flow:

### Iteration 4: Optimizing the Checkout API Route

**Before** (slow, sequential operations):

```typescript
// app/api/checkout/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { stripe } from '@/lib/stripe';
import { db } from '@/lib/db';
import { sendOrderConfirmationEmail } from '@/lib/email';

export async function POST(request: NextRequest) {
  const session = await getServerSession();
  if (!session?.user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { items } = await request.json();

  // Sequential operations - SLOW
  const order = await db.order.create({
    data: {
      userId: session.user.id,
      items: { create: items },
      status: 'pending',
    },
  });

  const paymentIntent = await stripe.paymentIntents.create({
    amount: calculateTotal(items),
    currency: 'usd',
    metadata: { orderId: order.id },
  });

  await db.order.update({
    where: { id: order.id },
    data: { stripePaymentIntentId: paymentIntent.id },
  });

  // This can take 3-5 seconds
  await sendOrderConfirmationEmail(session.user.email, order);

  return NextResponse.json({ clientSecret: paymentIntent.client_secret });
}
```

**Problem**: Each `await` blocks the next operation. Total time: 2s (DB) + 1s (Stripe) + 1s (DB update) + 4s (email) = 8+ seconds.

**After** (parallel operations, deferred email):

```typescript
// app/api/checkout/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { stripe } from '@/lib/stripe';
import { db } from '@/lib/db';

export async function POST(request: NextRequest) {
  const session = await getServerSession();
  if (!session?.user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { items } = await request.json();

  // Parallel operations - FAST
  const [order, paymentIntent] = await Promise.all([
    db.order.create({
      data: {
        userId: session.user.id,
        items: { create: items },
        status: 'pending',
      },
    }),
    stripe.paymentIntents.create({
      amount: calculateTotal(items),
      currency: 'usd',
      metadata: { userId: session.user.id },
    }),
  ]);

  // Update order with payment intent ID
  await db.order.update({
    where: { id: order.id },
    data: { stripePaymentIntentId: paymentIntent.id },
  });

  // Defer email to webhook handler (runs after payment succeeds)
  // No need to wait for email in the checkout request

  return NextResponse.json({ clientSecret: paymentIntent.client_secret });
}
```

```typescript
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { stripe } from '@/lib/stripe';
import { db } from '@/lib/db';
import { sendOrderConfirmationEmail } from '@/lib/email';

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = request.headers.get('stripe-signature')!;

  let event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 400 });
  }

  if (event.type === 'payment_intent.succeeded') {
    const paymentIntent = event.data.object;
    
    // Update order status
    const order = await db.order.update({
      where: { stripePaymentIntentId: paymentIntent.id },
      data: { status: 'paid' },
      include: { user: true },
    });

    // Send email asynchronously (webhook has 30s timeout)
    await sendOrderConfirmationEmail(order.user.email, order);
  }

  return NextResponse.json({ received: true });
}
```

**Improvement**: 
- Checkout API route: 2s (parallel DB + Stripe) + 1s (update) = 3 seconds total
- Email sending moved to webhook (runs after payment succeeds, not blocking checkout)
- User gets immediate response, email arrives shortly after

**Expected vs. Actual improvement**:
- **Before**: 8+ seconds, often timing out
- **After**: 3 seconds, well within 10-second limit
- **User experience**: Checkout completes instantly, confirmation email arrives within 30 seconds

**Verification**: Deploy and test checkout flow. Monitor Vercel function logs:

**Vercel Function Logs (After Optimization)**:
```
[INFO] POST /api/checkout - 200 OK (2847ms)
[INFO] POST /api/webhooks/stripe - 200 OK (4123ms)
```

### When to Apply This Solution

**What it optimizes for**:
- Fast API response times
- Staying within serverless function limits
- Better user experience (no waiting for emails)

**What it sacrifices**:
- Immediate email delivery (now asynchronous)
- Slightly more complex architecture (webhook handler required)

**When to choose this approach**:
- Any operation that takes >5 seconds
- Non-critical operations (emails, analytics, notifications)
- When using serverless functions with time limits

**When to avoid this approach**:
- Operations that must complete before responding (payment authorization)
- When you need guaranteed synchronous execution
- Simple applications where complexity isn't worth it

**Limitation preview**: This solves timeout issues, but we still haven't addressed image optimization and bundle size. Next, we'll tackle performance.

## Image optimization with next/image

## Phase 3: Image Optimization - The Performance Killer

Our application is deployed and functional. But let's check the performance:

**Lighthouse Report (Initial)**:
```
Performance: 42/100
- First Contentful Paint: 3.2s
- Largest Contentful Paint: 8.4s
- Cumulative Layout Shift: 0.45
- Total Blocking Time: 890ms

Opportunities:
- Properly size images: Potential savings of 2.4 MB
- Serve images in next-gen formats: Potential savings of 1.8 MB
- Eliminate render-blocking resources: 450ms
```

**Network Tab Analysis**:
```
Total resources: 45 requests
Total size: 4.2 MB
Images: 3.8 MB (90% of total)

product-1.jpg: 1.2 MB (3000x3000px, displayed at 300x300px)
product-2.jpg: 1.4 MB (3500x3500px, displayed at 300x300px)
product-3.jpg: 1.2 MB (3200x3200px, displayed at 300x300px)
```

### Diagnostic Analysis: Reading the Performance Failure

**Browser Behavior**:
- Page loads slowly
- Images pop in one by one
- Layout shifts as images load (content jumps around)
- Mobile users on slow connections wait 10+ seconds

**Lighthouse Evidence**:
- LCP (Largest Contentful Paint) of 8.4s is terrible (should be <2.5s)
- CLS (Cumulative Layout Shift) of 0.45 is poor (should be <0.1)
- Images are the primary bottleneck

**Network Tab Analysis**:
- Images are 90% of page weight
- Serving full-resolution images (3000x3000px) when only displaying 300x300px
- Using JPEG format (not optimized WebP/AVIF)
- No lazy loading (all images load immediately)

**Let's parse this evidence**:

1. **What the user experiences**: Slow page load, janky layout shifts, wasted bandwidth.

2. **What Lighthouse reveals**: Images are the performance killer. We're serving 10x more pixels than needed.

3. **What the Network tab shows**: Each image is 1+ MB. On a 3G connection, that's 4+ seconds per image.

4. **Root cause identified**: Using standard `<img>` tags (or unoptimized `<Image>` usage) without proper sizing, format optimization, or lazy loading.

5. **Why the current approach can't solve this**: Manual image optimization is tedious and error-prone. You'd need to create multiple sizes, convert formats, implement lazy loading—all by hand.

6. **What we need**: Next.js `<Image>` component with proper configuration to automatically optimize images.

### Iteration 5: Implementing Proper Image Optimization

Let's look at our current image usage:

**Before** (unoptimized):

```tsx
// components/ProductCard.tsx
'use client';

import Image from 'next/image';

export default function ProductCard({ product }: ProductCardProps) {
  return (
    <div className="border rounded-lg p-4">
      <Image
        src={product.imageUrl}
        alt={product.name}
        width={300}
        height={300}
        className="w-full h-48 object-cover rounded"
      />
      {/* ... rest of component */}
    </div>
  );
}
```

**Problem**: We're using `next/image`, but not optimally:
- No `sizes` attribute (Next.js doesn't know how large the image will be)
- No `priority` for above-the-fold images
- No `quality` setting
- Images might be external URLs (not optimized by Next.js)

**After** (optimized):

```tsx
// components/ProductCard.tsx
'use client';

import Image from 'next/image';

interface ProductCardProps {
  product: {
    id: string;
    name: string;
    price: number;
    imageUrl: string;
  };
  priority?: boolean; // For above-the-fold images
}

export default function ProductCard({ product, priority = false }: ProductCardProps) {
  return (
    <div className="border rounded-lg p-4">
      <Image
        src={product.imageUrl}
        alt={product.name}
        width={300}
        height={300}
        sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
        priority={priority}
        quality={75}
        className="w-full h-48 object-cover rounded"
      />
      {/* ... rest of component */}
    </div>
  );
}
```

**Improvements**:
1. **`sizes` attribute**: Tells Next.js the image will be:
   - 100% viewport width on mobile (<768px)
   - 50% viewport width on tablet (768-1200px)
   - 33% viewport width on desktop (>1200px)
   - Next.js generates appropriately sized images for each breakpoint

2. **`priority` prop**: First 3 images load immediately (above the fold), rest lazy load

3. **`quality={75}`**: Reduces file size by 30-40% with minimal visual difference

Now update the page to mark first images as priority:

```tsx
// app/products/page.tsx
import { db } from '@/lib/db';
import ProductCard from '@/components/ProductCard';

export default async function ProductsPage() {
  const products = await db.product.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' },
  });

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {products.map((product, index) => (
          <ProductCard
            key={product.id}
            product={product}
            priority={index < 3} // First 3 images load immediately
          />
        ))}
      </div>
    </div>
  );
}
```

### Configuring Next.js Image Optimization

If your images are hosted externally (e.g., on a CDN or cloud storage), you need to configure allowed domains:

```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'your-cdn.com',
        port: '',
        pathname: '/images/**',
      },
      {
        protocol: 'https',
        hostname: 'storage.googleapis.com',
        pathname: '/your-bucket/**',
      },
    ],
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
  },
};

module.exports = nextConfig;
```

**Configuration explained**:
- **`remotePatterns`**: Whitelist external image domains (security feature)
- **`formats`**: Serve AVIF (best compression) with WebP fallback
- **`deviceSizes`**: Breakpoints for responsive images
- **`imageSizes`**: Sizes for smaller images (icons, thumbnails)

Deploy and test:

**Lighthouse Report (After Optimization)**:
```
Performance: 89/100 ⬆️ +47 points
- First Contentful Paint: 1.1s ⬇️ -2.1s
- Largest Contentful Paint: 1.8s ⬇️ -6.6s
- Cumulative Layout Shift: 0.02 ⬇️ -0.43
- Total Blocking Time: 120ms ⬇️ -770ms

Opportunities:
- (No major opportunities remaining)
```

**Network Tab Analysis**:
```
Total resources: 45 requests
Total size: 420 KB ⬇️ -3.78 MB (90% reduction)

Images:
product-1.webp: 45 KB (300x300px, AVIF format)
product-2.webp: 52 KB (300x300px, AVIF format)
product-3.webp: 48 KB (300x300px, AVIF format)

Additional sizes generated:
product-1-640w.webp: 28 KB (for mobile)
product-1-1080w.webp: 65 KB (for tablet)
```

**Expected vs. Actual improvement**:
- **Bundle size**: 4.2 MB → 420 KB (90% reduction)
- **LCP**: 8.4s → 1.8s (79% faster)
- **CLS**: 0.45 → 0.02 (96% improvement)
- **User experience**: Page loads instantly, no layout shifts

### The Failure: Images Cause CLS (Cumulative Layout Shift)

Even with optimized images, you might still see layout shifts if you don't reserve space:

**Browser Behavior**:
- Page loads
- Text and layout appear
- Images pop in, pushing content down
- User tries to click a button, but it moves as image loads

**Lighthouse Evidence**:
```
Cumulative Layout Shift: 0.28
- Largest shift: 0.25 (caused by product image loading)
```

**React DevTools - Profiler**:
- Component re-renders when image loads
- Layout recalculation triggered

### Diagnostic Analysis: Reading the Layout Shift

**What the user experiences**: Content jumps around as images load. Frustrating, especially when trying to interact with the page.

**What Lighthouse reveals**: CLS score above 0.1 is poor. Images without reserved space cause layout shifts.

**Root cause identified**: Not providing explicit dimensions or aspect ratio for images.

**Solution**: Always provide `width` and `height` props to `<Image>`, or use aspect ratio CSS.

**Before** (causes CLS):

```tsx
<Image
  src={product.imageUrl}
  alt={product.name}
  fill // ❌ No dimensions, causes layout shift
  className="object-cover rounded"
/>
```

**After** (prevents CLS):

```tsx
<Image
  src={product.imageUrl}
  alt={product.name}
  width={300}
  height={300} // ✅ Explicit dimensions reserve space
  sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
  className="w-full h-48 object-cover rounded"
/>
```

**Or use aspect ratio**:

```tsx
<div className="relative aspect-square">
  <Image
    src={product.imageUrl}
    alt={product.name}
    fill // ✅ OK when parent has aspect ratio
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
    className="object-cover rounded"
  />
</div>
```

### When to Apply Image Optimization

**What it optimizes for**:
- Page load speed (smaller file sizes)
- Core Web Vitals (LCP, CLS)
- Bandwidth savings (especially mobile)
- Automatic format selection (AVIF/WebP)

**What it sacrifices**:
- Build time (image optimization during build)
- Server resources (on-demand optimization for dynamic images)
- Complexity (configuration required for external images)

**When to choose this approach**:
- Always, for any Next.js application with images
- Especially for image-heavy sites (e-commerce, portfolios, blogs)
- When Core Web Vitals matter (SEO, user experience)

**When to avoid this approach**:
- Never. There's no good reason to skip image optimization in Next.js.
- If you must use external image CDN with its own optimization, you can disable Next.js optimization:

```javascript
// next.config.js
const nextConfig = {
  images: {
    unoptimized: true, // Disable Next.js image optimization
  },
};
```

**Limitation preview**: Images are optimized, but we still haven't analyzed our JavaScript bundle size. Next, we'll use bundle analysis to identify bloat.

## Bundle analysis and Core Web Vitals

## Phase 4: Bundle Analysis - Finding the JavaScript Bloat

Images are optimized, but let's check our JavaScript bundle size:

**Build Output**:
```bash
Route (app)                              Size     First Load JS
┌ ○ /                                    142 B          287 kB
├ ○ /products                            12 kB          299 kB
├ ○ /products/[id]                       8.4 kB         295 kB
└ ○ /cart                                45 kB          332 kB

○  (Static)  prerendered as static content

First Load JS shared by all              287 kB
  ├ chunks/framework-abc123.js           45 kB
  ├ chunks/main-def456.js                32 kB
  ├ chunks/page-ghi789.js                210 kB  ⚠️ Large!
  └ other shared chunks (total)          0 B
```

### Diagnostic Analysis: Reading the Bundle Size Warning

**Build Output Evidence**:
- First Load JS: 287 kB (should be <100 kB for good performance)
- `page-ghi789.js`: 210 kB (73% of total bundle)
- Cart page: 332 kB total (45 kB page-specific + 287 kB shared)

**What This Means**:
- Every page loads 287 kB of JavaScript before it can become interactive
- The shared bundle is bloated—something large is being included on every page
- Cart page is especially heavy (332 kB total)

**Let's parse this evidence**:

1. **What the user experiences**: Slow Time to Interactive (TTI). Page looks loaded but buttons don't work yet.

2. **What the build output reveals**: Shared bundle is too large. Something is being included globally that shouldn't be.

3. **Root cause identified**: Likely a large library imported in a layout or root component, forcing it into the shared bundle.

4. **What we need**: Bundle analysis to identify the culprit, then code splitting to load it only where needed.

### Iteration 6: Installing Bundle Analyzer

Install the Next.js bundle analyzer:

```bash
npm install @next/bundle-analyzer
```

Configure it:

```javascript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'your-cdn.com',
        pathname: '/images/**',
      },
    ],
    formats: ['image/avif', 'image/webp'],
  },
};

module.exports = withBundleAnalyzer(nextConfig);
```

Run the analyzer:

```bash
ANALYZE=true npm run build
```

This opens an interactive visualization in your browser showing your bundle composition:

**Bundle Analyzer Output**:
```
Client Bundle Analysis:

Total: 287 kB

Breakdown:
- react + react-dom: 45 kB (16%)
- next/dist/client: 32 kB (11%)
- date-fns: 78 kB (27%) ⚠️ Large!
- lodash: 72 kB (25%) ⚠️ Large!
- framer-motion: 45 kB (16%)
- other: 15 kB (5%)
```

### Diagnostic Analysis: Identifying Bundle Bloat

**Bundle Analyzer Evidence**:
- `date-fns`: 78 kB (entire library imported)
- `lodash`: 72 kB (entire library imported)
- Together: 150 kB (52% of bundle)

**Investigation**: Where are these imported?

Search your codebase:

```bash
# Find date-fns imports
grep -r "from 'date-fns'" app/

# Output:
# app/layout.tsx:import { format } from 'date-fns';
# app/products/[id]/page.tsx:import { formatDistance } from 'date-fns';
```

**Root cause identified**: 
1. `date-fns` imported in `layout.tsx` (root layout) → included in shared bundle
2. Only using 2 functions (`format`, `formatDistance`) but importing entire library
3. Same issue with `lodash`

**Solution**: Use tree-shakeable imports and move imports to page level.

### Iteration 7: Optimizing Imports

**Before** (imports entire library):

```tsx
// app/layout.tsx
import { format } from 'date-fns'; // ❌ Imports entire date-fns library

export default function RootLayout({ children }: { children: React.ReactNode }) {
  const currentYear = format(new Date(), 'yyyy');
  
  return (
    <html lang="en">
      <body>
        {children}
        <footer>© {currentYear} Product Catalog</footer>
      </body>
    </html>
  );
}
```

**After** (tree-shakeable import):

```tsx
// app/layout.tsx
// ✅ Import only what you need
import { format } from 'date-fns/format';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  const currentYear = format(new Date(), 'yyyy');
  
  return (
    <html lang="en">
      <body>
        {children}
        <footer>© {currentYear} Product Catalog</footer>
      </body>
    </html>
  );
}
```

**Even better** (avoid library for simple cases):

```tsx
// app/layout.tsx
// ✅ No library needed for simple date formatting
export default function RootLayout({ children }: { children: React.ReactNode }) {
  const currentYear = new Date().getFullYear();
  
  return (
    <html lang="en">
      <body>
        {children}
        <footer>© {currentYear} Product Catalog</footer>
      </body>
    </html>
  );
}
```

**Fix lodash imports**:

**Before**:

```typescript
// lib/utils.ts
import _ from 'lodash'; // ❌ Imports entire lodash library (72 kB)

export function groupProductsByCategory(products: Product[]) {
  return _.groupBy(products, 'category');
}

export function uniqueCategories(products: Product[]) {
  return _.uniq(products.map(p => p.category));
}
```

**After**:

```typescript
// lib/utils.ts
// ✅ Import only specific functions
import groupBy from 'lodash/groupBy';
import uniq from 'lodash/uniq';

export function groupProductsByCategory(products: Product[]) {
  return groupBy(products, 'category');
}

export function uniqueCategories(products: Product[]) {
  return uniq(products.map(p => p.category));
}
```

**Or use native JavaScript** (best):

```typescript
// lib/utils.ts
// ✅ No library needed - use native JavaScript
export function groupProductsByCategory(products: Product[]) {
  return products.reduce((acc, product) => {
    const category = product.category;
    if (!acc[category]) acc[category] = [];
    acc[category].push(product);
    return acc;
  }, {} as Record<string, Product[]>);
}

export function uniqueCategories(products: Product[]) {
  return [...new Set(products.map(p => p.category))];
}
```

### Moving Heavy Imports to Client Components

If you must use a large library, load it only in Client Components where needed:

**Before** (in Server Component):

```tsx
// app/products/[id]/page.tsx - Server Component
import { formatDistance } from 'date-fns';

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.product.findUnique({ where: { id: params.id } });
  
  return (
    <div>
      <h1>{product.name}</h1>
      <p>Added {formatDistance(product.createdAt, new Date(), { addSuffix: true })}</p>
    </div>
  );
}
```

**After** (in Client Component):

```tsx
// app/products/[id]/page.tsx - Server Component
export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.product.findUnique({ where: { id: params.id } });
  
  return (
    <div>
      <h1>{product.name}</h1>
      <ProductTimestamp createdAt={product.createdAt} />
    </div>
  );
}
```

```tsx
// components/ProductTimestamp.tsx - Client Component
'use client';

import { formatDistance } from 'date-fns/formatDistance';

export default function ProductTimestamp({ createdAt }: { createdAt: Date }) {
  return (
    <p>
      Added {formatDistance(createdAt, new Date(), { addSuffix: true })}
    </p>
  );
}
```

**Improvement**: `date-fns` is now only loaded on pages that use `ProductTimestamp`, not in the shared bundle.

### Dynamic Imports for Heavy Components

For very large components or libraries, use dynamic imports:

```tsx
// app/products/[id]/page.tsx
import dynamic from 'next/dynamic';

// Dynamically import heavy component
const ProductReviews = dynamic(() => import('@/components/ProductReviews'), {
  loading: () => <p>Loading reviews...</p>,
  ssr: false, // Don't render on server (if not needed for SEO)
});

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.product.findUnique({ where: { id: params.id } });
  
  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      
      {/* Reviews component loads separately */}
      <ProductReviews productId={product.id} />
    </div>
  );
}
```

**Result**: `ProductReviews` and its dependencies load in a separate chunk, only when this page is visited.

### Rebuild and Analyze

After optimizations:

```bash
ANALYZE=true npm run build
```

**Build Output (After Optimization)**:
```bash
Route (app)                              Size     First Load JS
┌ ○ /                                    142 B          87 kB ⬇️ -200 kB
├ ○ /products                            12 kB          99 kB ⬇️ -200 kB
├ ○ /products/[id]                       8.4 kB         95 kB ⬇️ -200 kB
└ ○ /cart                                45 kB          132 kB ⬇️ -200 kB

First Load JS shared by all              87 kB ⬇️ -200 kB
  ├ chunks/framework-abc123.js           45 kB
  ├ chunks/main-def456.js                32 kB
  ├ chunks/page-ghi789.js                10 kB ⬇️ -200 kB
  └ other shared chunks (total)          0 B
```

**Bundle Analyzer Output (After)**:
```
Total: 87 kB ⬇️ -200 kB (70% reduction)

Breakdown:
- react + react-dom: 45 kB (52%)
- next/dist/client: 32 kB (37%)
- other: 10 kB (11%)
```

**Expected vs. Actual improvement**:
- **Shared bundle**: 287 kB → 87 kB (70% reduction)
- **First Load JS**: 287 kB → 87 kB (all pages improved)
- **Time to Interactive**: 2.8s → 1.2s (57% faster)

### Core Web Vitals: The Complete Picture

Now let's measure the full impact on Core Web Vitals:

**Lighthouse Report (Final)**:
```
Performance: 96/100 ⬆️ +7 points

Core Web Vitals:
- LCP (Largest Contentful Paint): 1.2s ✅ (Good: <2.5s)
- FID (First Input Delay): 45ms ✅ (Good: <100ms)
- CLS (Cumulative Layout Shift): 0.01 ✅ (Good: <0.1)

Additional Metrics:
- First Contentful Paint: 0.8s
- Time to Interactive: 1.2s
- Speed Index: 1.4s
- Total Blocking Time: 80ms

Passed Audits:
✅ Properly size images
✅ Serve images in next-gen formats
✅ Eliminate render-blocking resources
✅ Minimize main-thread work
✅ Reduce JavaScript execution time
```

**Real User Monitoring (RUM) Data**:
```
75th Percentile (Real Users):
- LCP: 1.8s (Good)
- FID: 65ms (Good)
- CLS: 0.03 (Good)

Device Breakdown:
- Desktop: LCP 1.2s, FID 35ms, CLS 0.02
- Mobile (4G): LCP 2.1s, FID 85ms, CLS 0.04
- Mobile (3G): LCP 3.8s, FID 120ms, CLS 0.05
```

### When to Apply Bundle Optimization

**What it optimizes for**:
- Time to Interactive (TTI)
- First Input Delay (FID)
- JavaScript execution time
- Mobile performance

**What it sacrifices**:
- Development time (analyzing and optimizing)
- Code organization (sometimes need to split components)
- Convenience (can't just `import _ from 'lodash'`)

**When to choose this approach**:
- When First Load JS exceeds 100 kB
- When Lighthouse shows "Reduce JavaScript execution time"
- When Time to Interactive is >3 seconds
- Before launching to production

**When to avoid this approach**:
- During early development (premature optimization)
- When bundle size is already <100 kB
- When the library is truly needed everywhere (e.g., React itself)

**Code characteristics**:
- Setup: Medium (install analyzer, configure)
- Maintenance: Low (once optimized, stays optimized)
- Performance impact: High (70% bundle reduction in our case)

**Limitation preview**: Bundle is optimized, but we haven't discussed runtime choice. Next, we'll explore Edge vs. Node.js runtime.

## Edge runtime vs. Node.js runtime

## Phase 5: Runtime Choice - Edge vs. Node.js

Next.js supports two runtime environments:
1. **Node.js runtime** (default): Full Node.js API, runs on Vercel's regional servers
2. **Edge runtime**: Lightweight, runs on Vercel's global edge network

Let's understand when to use each.

### The Default: Node.js Runtime

Our application currently uses Node.js runtime (the default). Let's see what that means:

**Current Deployment Architecture**:
```
User Request (Tokyo)
    ↓
Vercel Edge Network (Tokyo) - CDN only
    ↓
Vercel Regional Server (US East) - Node.js runtime
    ↓ (200ms latency)
Database (US East)
    ↓
Response travels back to Tokyo (200ms)
    ↓
Total: ~400ms + processing time
```

**Vercel Function Logs (Node.js Runtime)**:
```
[INFO] GET /products - Region: iad1 (US East)
[INFO] Cold start: 450ms
[INFO] Database query: 120ms
[INFO] Total: 570ms
```

**User in Tokyo**:
```
Request timing:
- DNS lookup: 20ms
- Connection: 180ms
- Waiting (TTFB): 570ms ← Server processing
- Content download: 50ms
Total: 820ms
```

### The Alternative: Edge Runtime

Edge runtime runs your code on Vercel's global edge network—closer to users. But it has limitations.

### Iteration 8: Converting to Edge Runtime

Let's try converting our product listing page to Edge runtime:

**Before** (Node.js runtime):

```tsx
// app/products/page.tsx
import { db } from '@/lib/db';

export default async function ProductsPage() {
  const products = await db.product.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' },
  });

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {products.map((product) => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  );
}
```

**After** (attempting Edge runtime):

```tsx
// app/products/page.tsx
import { db } from '@/lib/db';

export const runtime = 'edge'; // ← Enable Edge runtime

export default async function ProductsPage() {
  const products = await db.product.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' },
  });

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {products.map((product) => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  );
}
```

Deploy and observe:

**Terminal Output (Build Failure)**:
```bash
[12:34:56.789] Running build...
[12:34:58.123] 
[12:34:58.123] Error: The Edge Runtime does not support Node.js 'crypto' module.
[12:34:58.123] 
[12:34:58.123] Import trace for requested module:
[12:34:58.123] ./node_modules/@prisma/client/runtime/library.js
[12:34:58.123] ./node_modules/@prisma/client/default.js
[12:34:58.123] ./lib/db.ts
[12:34:58.123] ./app/products/page.tsx
[12:34:58.123] 
[12:34:58.456] BUILD FAILED
```

### Diagnostic Analysis: Edge Runtime Limitations

**Build Failure Evidence**:
- Error: "The Edge Runtime does not support Node.js 'crypto' module"
- Import trace shows Prisma Client uses Node.js APIs
- Build fails before deployment

**What Happened**:
Edge runtime is a lightweight JavaScript runtime (based on V8, like browsers). It doesn't include full Node.js APIs. Prisma Client uses Node.js-specific modules (`crypto`, `fs`, `net`) that aren't available in Edge runtime.

**Root cause identified**: Not all libraries work in Edge runtime. Prisma, most ORMs, and many Node.js libraries require the full Node.js runtime.

**What works in Edge runtime**:
- Fetch API
- Web Crypto API
- Lightweight libraries (no Node.js dependencies)
- Simple data transformations

**What doesn't work in Edge runtime**:
- Prisma (uses Node.js crypto, fs)
- Most database drivers (use Node.js net)
- File system operations
- Node.js built-in modules

### When to Use Edge Runtime

Edge runtime is best for:
1. **API routes that don't need database access**
2. **Middleware** (authentication checks, redirects)
3. **Static content with dynamic headers**
4. **Geolocation-based responses**

Let's see a valid use case:

### Iteration 9: Edge Runtime for Geolocation API

Create a geolocation API route that benefits from Edge runtime:

```typescript
// app/api/geo/route.ts
export const runtime = 'edge'; // ✅ Perfect for Edge runtime

export async function GET(request: Request) {
  // Access geolocation from request headers (provided by Vercel Edge)
  const country = request.headers.get('x-vercel-ip-country') || 'Unknown';
  const city = request.headers.get('x-vercel-ip-city') || 'Unknown';
  const region = request.headers.get('x-vercel-ip-country-region') || 'Unknown';

  return Response.json({
    country,
    city,
    region,
    message: `Hello from ${city}, ${country}!`,
  });
}
```

**Deployment Architecture (Edge Runtime)**:
```
User Request (Tokyo)
    ↓
Vercel Edge Network (Tokyo) - Edge runtime executes here
    ↓ (5ms latency)
Response immediately
    ↓
Total: ~5ms + processing time
```

**Vercel Function Logs (Edge Runtime)**:
```
[INFO] GET /api/geo - Region: nrt1 (Tokyo)
[INFO] Cold start: 0ms (Edge runtime is always warm)
[INFO] Total: 5ms
```

**User in Tokyo**:
```
Request timing:
- DNS lookup: 20ms
- Connection: 5ms
- Waiting (TTFB): 5ms ← Edge runtime
- Content download: 5ms
Total: 35ms (vs. 820ms with Node.js runtime)
```

**Expected vs. Actual improvement**:
- **Latency**: 820ms → 35ms (96% reduction)
- **Cold start**: 450ms → 0ms (Edge is always warm)
- **Global performance**: Consistent low latency worldwide

### Edge Runtime for Middleware

Middleware is the perfect use case for Edge runtime:

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

// Middleware automatically runs on Edge runtime
export function middleware(request: NextRequest) {
  // Check authentication
  const token = request.cookies.get('auth-token');
  
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    // Redirect to login
    return NextResponse.redirect(new URL('/login', request.url));
  }

  // Add custom headers
  const response = NextResponse.next();
  response.headers.set('x-custom-header', 'value');
  
  return response;
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*'],
};
```

**Why Edge runtime is perfect for middleware**:
- Runs before every request (needs to be fast)
- No database access needed (just checking cookies/headers)
- Benefits from global distribution
- Minimal cold start time

### Hybrid Approach: Edge + Node.js

The best architecture uses both runtimes strategically:

**Optimal Architecture**:
```
┌─────────────────────────────────────────────────────────────┐
│ Edge Runtime (Global)                                       │
│ - Middleware (auth checks, redirects)                       │
│ - Geolocation API                                           │
│ - Static content with dynamic headers                       │
│ - A/B testing logic                                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Node.js Runtime (Regional)                                  │
│ - Database queries (Prisma)                                 │
│ - Complex business logic                                    │
│ - File uploads                                              │
│ - Third-party API calls                                     │
└─────────────────────────────────────────────────────────────┘
```

### Example: Hybrid Product Listing

Keep database queries in Node.js runtime, but add Edge middleware for caching:

```typescript
// middleware.ts (Edge runtime)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Check if we have cached products
  const cached = request.cookies.get('products-cache');
  
  if (cached) {
    // Return cached response immediately from Edge
    return new NextResponse(cached.value, {
      headers: {
        'Content-Type': 'application/json',
        'X-Cache': 'HIT',
      },
    });
  }

  // No cache, continue to Node.js runtime
  return NextResponse.next();
}

export const config = {
  matcher: '/api/products',
};
```

```typescript
// app/api/products/route.ts (Node.js runtime - default)
import { db } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET() {
  const products = await db.product.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' },
  });

  const response = NextResponse.json(products);
  
  // Set cache cookie for Edge middleware
  response.cookies.set('products-cache', JSON.stringify(products), {
    maxAge: 60, // 1 minute cache
  });

  return response;
}
```

**Result**:
- First request: Edge → Node.js → Database (570ms)
- Subsequent requests (within 1 minute): Edge only (5ms)
- 99% of requests served from Edge with 5ms latency

### When to Apply Edge Runtime

**What it optimizes for**:
- Global latency (consistent performance worldwide)
- Cold start time (Edge is always warm)
- Simple, fast operations

**What it sacrifices**:
- Node.js API access (no Prisma, no fs, no crypto)
- Library compatibility (many npm packages won't work)
- Debugging complexity (different runtime environment)

**When to choose Edge runtime**:
- Middleware (always)
- Geolocation-based responses
- Simple API routes without database access
- A/B testing logic
- Authentication checks (token validation, not database queries)

**When to avoid Edge runtime**:
- Database queries (use Node.js runtime)
- File uploads
- Complex business logic with Node.js dependencies
- Third-party libraries that use Node.js APIs

**Code characteristics**:
- Setup: Easy (add `export const runtime = 'edge'`)
- Maintenance: Low (once working, stays working)
- Performance impact: High (96% latency reduction for suitable use cases)
- Compatibility: Limited (many libraries won't work)

**Decision Framework**:

| Use Case | Runtime | Reason |
|----------|---------|--------|
| Database queries | Node.js | Prisma requires Node.js |
| Middleware | Edge | Fast, global, no DB needed |
| File uploads | Node.js | Requires fs module |
| Geolocation API | Edge | Benefits from global distribution |
| Payment processing | Node.js | Complex logic, third-party SDKs |
| A/B testing | Edge | Fast decision, no DB needed |
| Email sending | Node.js | SMTP libraries need Node.js |
| JWT validation | Edge | Fast, no DB needed |

## The Complete Journey - Deployment to Production

## Phase 6: Synthesis - The Deployment Journey

Let's review the complete evolution of our production deployment:

### The Journey: From Broken Build to Optimized Production

| Iteration | Failure Mode | Technique Applied | Result | Performance Impact |
|-----------|--------------|-------------------|--------|-------------------|
| 0 | Build fails (Prisma types missing) | Add `postinstall` script | Build succeeds | N/A |
| 1 | Runtime error (can't connect to localhost DB) | Configure production env vars | App loads | N/A |
| 2 | Images 404, API routes fail | Commit images, set env vars for all environments | Fully functional | N/A |
| 3 | Checkout times out (10s limit) | Optimize API route, use webhooks | Fast checkout | 8s → 3s |
| 4 | Poor performance (LCP 8.4s) | Optimize images with next/image | Fast page loads | LCP 8.4s → 1.8s |
| 5 | Large bundle (287 kB) | Bundle analysis, tree-shaking, dynamic imports | Small bundle | 287 kB → 87 kB |
| 6 | High latency for global users | Edge runtime for middleware/geo API | Low latency | 820ms → 35ms |

### Final Implementation: Production-Ready Deployment

Here's our complete, optimized production setup:

```json
// package.json
{
  "name": "product-catalog",
  "version": "1.0.0",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "postinstall": "prisma generate"
  },
  "dependencies": {
    "@prisma/client": "^5.7.0",
    "next": "14.0.0",
    "next-auth": "^4.24.0",
    "react": "^18.2.0",
    "stripe": "^14.0.0"
  },
  "devDependencies": {
    "@next/bundle-analyzer": "^14.0.0",
    "prisma": "^5.7.0",
    "typescript": "^5.3.0"
  }
}
```

```javascript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'your-cdn.com',
        pathname: '/images/**',
      },
    ],
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
  },
};

module.exports = withBundleAnalyzer(nextConfig);
```

```typescript
// middleware.ts (Edge runtime)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Authentication check
  const token = request.cookies.get('auth-token');
  
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/dashboard/:path*'],
};
```

```tsx
// app/products/page.tsx (Node.js runtime - default)
import { db } from '@/lib/db';
import ProductCard from '@/components/ProductCard';

export default async function ProductsPage() {
  const products = await db.product.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' },
  });

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Our Products</h1>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {products.map((product, index) => (
          <ProductCard
            key={product.id}
            product={product}
            priority={index < 3}
          />
        ))}
      </div>
    </div>
  );
}
```

```tsx
// components/ProductCard.tsx
'use client';

import Image from 'next/image';

interface ProductCardProps {
  product: {
    id: string;
    name: string;
    price: number;
    imageUrl: string;
  };
  priority?: boolean;
}

export default function ProductCard({ product, priority = false }: ProductCardProps) {
  return (
    <div className="border rounded-lg p-4">
      <Image
        src={product.imageUrl}
        alt={product.name}
        width={300}
        height={300}
        sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
        priority={priority}
        quality={75}
        className="w-full h-48 object-cover rounded"
      />
      <h3 className="text-xl font-semibold mt-4">{product.name}</h3>
      <p className="text-gray-600">${product.price}</p>
    </div>
  );
}
```

```typescript
// app/api/checkout/route.ts (Node.js runtime - default)
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { stripe } from '@/lib/stripe';
import { db } from '@/lib/db';

export async function POST(request: NextRequest) {
  const session = await getServerSession();
  if (!session?.user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { items } = await request.json();

  // Parallel operations
  const [order, paymentIntent] = await Promise.all([
    db.order.create({
      data: {
        userId: session.user.id,
        items: { create: items },
        status: 'pending',
      },
    }),
    stripe.paymentIntents.create({
      amount: calculateTotal(items),
      currency: 'usd',
      metadata: { userId: session.user.id },
    }),
  ]);

  await db.order.update({
    where: { id: order.id },
    data: { stripePaymentIntentId: paymentIntent.id },
  });

  return NextResponse.json({ clientSecret: paymentIntent.client_secret });
}

function calculateTotal(items: any[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0) * 100;
}
```

```typescript
// app/api/geo/route.ts (Edge runtime)
export const runtime = 'edge';

export async function GET(request: Request) {
  const country = request.headers.get('x-vercel-ip-country') || 'Unknown';
  const city = request.headers.get('x-vercel-ip-city') || 'Unknown';

  return Response.json({
    country,
    city,
    message: `Hello from ${city}, ${country}!`,
  });
}
```

### Environment Variables Configuration

**Vercel Dashboard → Project Settings → Environment Variables**:

| Variable | Production | Preview | Development |
|----------|-----------|---------|-------------|
| `DATABASE_URL` | `postgres://prod...` | `postgres://preview...` | `postgres://localhost...` |
| `NEXTAUTH_SECRET` | `[prod-secret]` | `[preview-secret]` | `[dev-secret]` |
| `NEXTAUTH_URL` | `https://app.com` | `https://preview.vercel.app` | `http://localhost:3000` |
| `STRIPE_SECRET_KEY` | `sk_live_...` | `sk_test_...` | `sk_test_...` |
| `STRIPE_PUBLISHABLE_KEY` | `pk_live_...` | `pk_test_...` | `pk_test_...` |
| `STRIPE_WEBHOOK_SECRET` | `whsec_prod...` | `whsec_preview...` | `whsec_local...` |

### Performance Metrics: Before vs. After

**Initial Deployment (Iteration 0-3)**:
```
Performance: 42/100
- LCP: 8.4s
- FID: 450ms
- CLS: 0.45
- Bundle size: 287 kB
- Image size: 3.8 MB
- TTFB (Tokyo): 820ms
```

**Optimized Production (Iteration 4-6)**:
```
Performance: 96/100 ⬆️ +54 points
- LCP: 1.2s ⬇️ -7.2s (86% improvement)
- FID: 45ms ⬇️ -405ms (90% improvement)
- CLS: 0.01 ⬇️ -0.44 (98% improvement)
- Bundle size: 87 kB ⬇️ -200 kB (70% reduction)
- Image size: 420 KB ⬇️ -3.38 MB (89% reduction)
- TTFB (Tokyo): 35ms ⬇️ -785ms (96% improvement)
```

### Decision Framework: Deployment Checklist

Before deploying to production, verify:

**Build Configuration**:
- [ ] `postinstall` script for Prisma generation
- [ ] Bundle analyzer configured
- [ ] Image optimization configured
- [ ] TypeScript strict mode enabled

**Environment Variables**:
- [ ] All secrets set in Vercel dashboard
- [ ] Different values for Production/Preview/Development
- [ ] Database URL points to production database
- [ ] API keys are production keys (not test keys)

**Performance**:
- [ ] Lighthouse score >90
- [ ] LCP <2.5s
- [ ] FID <100ms
- [ ] CLS <0.1
- [ ] First Load JS <100 kB
- [ ] Images optimized with next/image

**Runtime**:
- [ ] Middleware uses Edge runtime
- [ ] Database queries use Node.js runtime
- [ ] API routes optimized for <10s execution
- [ ] Long operations moved to webhooks/background jobs

**Security**:
- [ ] Environment variables not committed to Git
- [ ] `.env.local` in `.gitignore`
- [ ] Production secrets different from development
- [ ] API routes have authentication checks

**Monitoring**:
- [ ] Error tracking configured (Sentry, etc.)
- [ ] Analytics configured
- [ ] Core Web Vitals monitoring enabled
- [ ] Vercel function logs reviewed

### Common Production Failures and Their Signatures

**Symptom: Build fails with "Property does not exist on type"**

**Console pattern**:
```
Type error: Property 'product' does not exist on type 'PrismaClient'
```

**Root cause**: Prisma Client not generated
**Solution**: Add `postinstall` script

---

**Symptom: Runtime error "Can't reach database server"**

**Function logs**:
```
PrismaClientInitializationError: Can't reach database server at `localhost:5432`
```

**Root cause**: Missing or incorrect `DATABASE_URL` environment variable
**Solution**: Set production database URL in Vercel dashboard

---

**Symptom: API route times out (504 Gateway Timeout)**

**Function logs**:
```
Task timed out after 10.00 seconds
```

**Root cause**: Operation takes longer than serverless function limit
**Solution**: Optimize with parallel operations, move to webhooks, or upgrade plan

---

**Symptom: Images return 404**

**Console pattern**:
```
GET /images/product-1.jpg 404 (Not Found)
```

**Root cause**: Images not committed to Git or in `.gitignore`
**Solution**: Remove images from `.gitignore`, commit and push

---

**Symptom: Poor performance, high LCP**

**Lighthouse evidence**:
```
LCP: 8.4s
Opportunities: Properly size images (2.4 MB savings)
```

**Root cause**: Unoptimized images
**Solution**: Use `next/image` with proper configuration

---

**Symptom: Large bundle size, slow TTI**

**Build output**:
```
First Load JS: 287 kB
chunks/page-ghi789.js: 210 kB
```

**Root cause**: Large libraries in shared bundle
**Solution**: Bundle analysis, tree-shaking, dynamic imports

---

**Symptom: High latency for global users**

**Function logs**:
```
Region: iad1 (US East)
Total: 570ms
```

**Root cause**: All requests go to single region
**Solution**: Use Edge runtime for suitable routes

### Lessons Learned

**1. Environment Variables Are Not Magic**
- `.env.local` files don't deploy automatically
- Each environment (Production/Preview/Development) needs separate configuration
- Always use different secrets for production vs. development

**2. Serverless Has Limits**
- 10-second timeout on free tier
- Optimize for fast execution
- Move long operations to webhooks or background jobs

**3. Images Are Usually the Bottleneck**
- Unoptimized images can be 90% of page weight
- `next/image` is not optional—it's essential
- Always provide dimensions to prevent CLS

**4. Bundle Size Matters**
- Every kilobyte of JavaScript delays interactivity
- Tree-shaking and dynamic imports are your friends
- Analyze before optimizing (don't guess)

**5. Runtime Choice Is Strategic**
- Edge runtime: Fast, global, limited
- Node.js runtime: Full-featured, regional
- Use both strategically for optimal performance

**6. Deployment Is Iterative**
- First deployment will likely fail
- Each failure teaches you something
- Build → Deploy → Measure → Optimize → Repeat

**7. Performance Is a Feature**
- Core Web Vitals affect SEO and user experience
- Lighthouse scores correlate with conversion rates
- Optimize before launch, not after

The journey from broken build to optimized production is not linear. Each failure reveals a new layer of complexity. But by systematically diagnosing each issue, applying the right technique, and measuring the impact, you build a production-ready application that performs well for users worldwide.
