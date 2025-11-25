# Chapter 26: Going to Production

## Environment configuration

## The Final Mile: From Localhost to Live

So far, we've built, tested, and optimized our application in a development environment. But `localhost:3000` is a safe, predictable bubble. The real world—production—is messy. It involves different databases, secret API keys, third-party services, and the terrifying prospect of real users encountering real bugs.

This chapter is about bridging that gap. We'll take a simple, functional component and harden it for production, introducing the essential practices that separate hobby projects from professional, scalable applications.

### Phase 1: Establish the Reference Implementation

Our anchor example for this chapter will be a `ProductDetailsPage`. It's a common, realistic component that fetches data and displays it. This component will be our patient, and we will perform a series of production-hardening procedures on it.

Here is its initial, naive implementation. It works perfectly on a developer's machine, but it's a ticking time bomb in a real deployment pipeline.

**Project Structure**:
```
src/
└── app/
    └── product/
        └── [productId]/
            └── page.tsx      ← Our reference implementation
```

```tsx
// src/app/product/[productId]/page.tsx

import { notFound } from 'next/navigation';

type Product = {
  id: string;
  name: string;
  price: number;
  description: string;
};

async function getProduct(productId: string): Promise<Product | null> {
  // DANGER: Hardcoded API endpoint
  const res = await fetch(`http://api.local/products/${productId}`);
  
  if (!res.ok) {
    // This will be handled by the nearest error.js file
    // For now, let's assume it might just fail silently or return null
    return null;
  }
  return res.json();
}

export default async function ProductDetailsPage({ params }: { params: { productId: string } }) {
  const product = await getProduct(params.productId);

  if (!product) {
    notFound();
  }

  return (
    <main>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <strong>${product.price.toFixed(2)}</strong>
      <button>Add to Cart</button>
    </main>
  );
}
```

### The First Failure: The Hardcoded API Endpoint

Our component works, but it has a fatal flaw: `const res = await fetch('http://api.local/products/...')`.

**Current Limitation**: This URL is hardcoded. This is fine for local development where we might be running a mock API server at `api.local`. But what happens when we deploy?

**New Scenario Introduction**: We need to deploy this application to two environments:
1.  **Staging**: A pre-production environment for QA testing. Its API is at `https://api.staging.my-app.com`.
2.  **Production**: The live site for real users. Its API is at `https://api.my-app.com`.

With the current code, we would have to manually change the URL in `page.tsx`, commit, and deploy for each environment. This is error-prone, inefficient, and a guaranteed way to cause a production outage by accidentally pointing the live site to the staging API.

This isn't a "bug" that throws an error; it's an architectural failure. The code is not portable across environments.

### Technique Introduced: Environment Variables

To solve this, we must decouple configuration from code. The standard mechanism for this is **environment variables**. Next.js has built-in support for them using `.env` files.

**How it works**:
- You create files like `.env.local`, `.env.development`, `.env.production`.
- Next.js loads the appropriate file based on the current environment (`npm run dev` vs. `npm run build`).
- These variables are exposed to your application via `process.env`.

**CRITICAL: Server vs. Browser Variables**
By default, all variables defined in `.env` files are only available on the server (e.g., in Server Components, `getStaticProps`, `getServerSideProps`, API routes). This is a security feature to prevent leaking secret keys to the client.

To expose a variable to the browser, you **must** prefix it with `NEXT_PUBLIC_`.

Our `ProductDetailsPage` is a Server Component, so it runs on the server. We don't strictly *need* the `NEXT_PUBLIC_` prefix for the API URL. However, it's a common pattern to fetch data on the client side as well. To keep our configuration consistent and flexible, we'll adopt the `NEXT_PUBLIC_` prefix as a good practice for non-secret URLs.

### Solution Implementation: Using Environment Variables

Let's create our environment configuration files.

**Step 1: Create `.env` files**
These files should be in the root of your project and **added to `.gitignore`** (except for `.env.example`).

```bash
# .env.development.local (for `npm run dev`)
NEXT_PUBLIC_API_URL="http://api.local"
```
```bash
# .env.production.local (for `npm run build` and `npm run start`)
# In a real project, these values would be set by your hosting provider (Vercel, Netlify, AWS).
NEXT_PUBLIC_API_URL="https://api.my-app.com"
```
```bash
# .gitignore
.env*.local
```

**Step 2: Refactor the component**

Now, we'll update our `getProduct` function to use the new environment variable.

**Before**:
```typescript
// src/app/product/[productId]/page.tsx (snippet)
async function getProduct(productId: string): Promise<Product | null> {
  // DANGER: Hardcoded API endpoint
  const res = await fetch(`http://api.local/products/${productId}`);
  // ...
}
```

**After**:
```typescript
// src/app/product/[productId]/page.tsx (snippet)
async function getProduct(productId: string): Promise<Product | null> {
  // ✅ Reads configuration from the environment
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;
  const res = await fetch(`${apiUrl}/products/${productId}`);
  // ...
}
```

### Verification

How do we know it's working?

1.  **Run in Development**:
    -   Start the dev server: `npm run dev`.
    -   In the terminal where you ran the command, Next.js will log which `.env` files were loaded.
    -   When you visit a product page, the `fetch` call will go to `http://api.local/products/...`.

2.  **Run in Production Mode (Locally)**:
    -   Build the app: `npm run build`.
    -   Start the production server: `npm run start`.
    -   When you visit a product page, the `fetch` call will now go to `https://api.my-app.com/products/...`.

**Expected vs. Actual Improvement**:
- **Expected**: The application should point to different APIs depending on the environment it's running in, without any code changes.
- **Actual**: The application now correctly separates configuration from code, making it portable and safe to deploy across multiple environments.

**Limitation Preview**: Our configuration is now flexible, but our feature set is rigid. What if we want to ship code for a new feature but not show it to all users immediately? This leads us to feature flags.

## Feature flags with simple patterns

## Iteration 1: Controlling Feature Rollouts

Our component now correctly fetches data from different environments. But in modern software development, "shipping" and "releasing" are two separate events. We want to be able to merge and deploy code to production *without* making it visible to all users.

**Current state recap**: Our `ProductDetailsPage` shows product information.
**Current limitation**: Any new UI we add to the component is immediately live for everyone upon deployment.

**New scenario introduction**: The product team wants to add a "Customer Reviews" section to the page. However, the backend API for reviews isn't ready yet, and they want to release it to internal employees first for testing before a public launch.

### The Failure: All-or-Nothing Deployments

The naive approach is to just build the new component and add it to the page.

Let's create a placeholder `CustomerReviews` component.

**Project Structure**:
```
src/
├── app/
│   └── product/
│       └── [productId]/
│           └── page.tsx
└── components/
    └── CustomerReviews.tsx   ← New component
```

```tsx
// src/components/CustomerReviews.tsx

// A placeholder component for now.
async function getReviews(productId: string) {
  console.log(`Fetching reviews for ${productId}...`);
  // Simulate network delay
  await new Promise(resolve => setTimeout(resolve, 500));
  return [
    { id: 1, author: 'Jane Doe', comment: 'Amazing product!' },
    { id: 2, author: 'John Smith', comment: 'Could be better.' },
  ];
}

export default async function CustomerReviews({ productId }: { productId: string }) {
  const reviews = await getReviews(productId);

  return (
    <div style={{ marginTop: '2rem', borderTop: '1px solid #ccc', paddingTop: '1rem' }}>
      <h2>Customer Reviews</h2>
      <ul>
        {reviews.map(review => (
          <li key={review.id}>
            <strong>{review.author}</strong>: {review.comment}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

Now, let's add it to our page.

```tsx
// src/app/product/[productId]/page.tsx (Updated)

import { notFound } from 'next/navigation';
import CustomerReviews from '@/components/CustomerReviews'; // ← Import

// ... (getProduct function is the same)

export default async function ProductDetailsPage({ params }: { params: { productId: string } }) {
  const product = await getProduct(params.productId);

  if (!product) {
    notFound();
  }

  return (
    <main>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <strong>${product.price.toFixed(2)}</strong>
      <button>Add to Cart</button>

      {/* The new feature is now live for everyone */}
      <CustomerReviews productId={params.productId} />
    </main>
  );
}
```

### Diagnostic Analysis: Reading the Failure

This isn't a code failure; it's a process failure.

**Browser Behavior**: Every single user who visits the product page after this deployment will see the "Customer Reviews" section.

**Business Impact**:
- If the backend API isn't ready, the component might crash or show an error, degrading the user experience.
- We can't get feedback from a small group of users before a full rollout.
- If we find a bug in the new feature, we have to do a full rollback/hotfix deployment, which is stressful and slow.

**Let's parse this evidence**:

1.  **What the user experiences**: A new feature appears, whether it's ready or not.
2.  **What the console reveals**: Nothing is wrong from a technical perspective. The code runs as written.
3.  **Root cause identified**: The visibility of a feature is directly tied to its presence in the codebase.
4.  **Why the current approach can't solve this**: We have no mechanism to control UI visibility at runtime based on configuration.
5.  **What we need**: A way to conditionally render the `CustomerReviews` component based on a flag that we can change without a new deployment. This is a **feature flag** (or feature toggle).

### Technique Introduced: Feature Flags via Environment Variables

For simple on/off toggles, we can reuse the environment variable system we just learned. This is a powerful pattern for enabling and disabling features during a rollout.

### Solution Implementation: Adding a Feature Flag

**Step 1: Add the feature flag to your `.env` files**

We'll add a new variable, `NEXT_PUBLIC_FEATURE_REVIEWS_ENABLED`. We set it to `true` in development but would keep it `false` in the production environment file until we're ready to launch.

```bash
# .env.development.local
NEXT_PUBLIC_API_URL="http://api.local"
NEXT_PUBLIC_FEATURE_REVIEWS_ENABLED="true"
```
```bash
# .env.production.local
NEXT_PUBLIC_API_URL="https://api.my-app.com"
NEXT_PUBLIC_FEATURE_REVIEWS_ENABLED="false" # Off in production by default
```

**Step 2: Conditionally render the component**

Now we update the page to check this flag.

**Before**:
```tsx
// src/app/product/[productId]/page.tsx (snippet)
// ...
import CustomerReviews from '@/components/CustomerReviews';

export default async function ProductDetailsPage({ params }) {
  // ...
  return (
    <main>
      {/* ... product details ... */}
      <CustomerReviews productId={params.productId} />
    </main>
  );
}
```

**After**:
```tsx
// src/app/product/[productId]/page.tsx (snippet)
// ...
import CustomerReviews from '@/components/CustomerReviews';

export default async function ProductDetailsPage({ params }: { params: { productId: string } }) {
  const product = await getProduct(params.productId);
  
  // ✅ Read the feature flag from the environment
  const reviewsEnabled = process.env.NEXT_PUBLIC_FEATURE_REVIEWS_ENABLED === 'true';

  if (!product) {
    notFound();
  }

  return (
    <main>
      {/* ... product details ... */}
      
      {/* ✅ Conditionally render the new feature */}
      {reviewsEnabled && <CustomerReviews productId={params.productId} />}
    </main>
  );
}
```

### Verification

Now, when we run the application:
-   `npm run dev`: The `CustomerReviews` component will be visible because `NEXT_PUBLIC_FEATURE_REVIEWS_ENABLED` is `"true"` in `.env.development.local`.
-   `npm run build` & `npm run start`: The component will be hidden, because the flag is `"false"` in `.env.production.local`.

To release the feature, we would simply update the environment variable in our production hosting environment (e.g., on Vercel's dashboard) from `false` to `true`. The change would take effect on the next deployment, or immediately if our provider supports it, without any code changes.

### When to Apply This Solution

-   **What it optimizes for**: Decoupling deployment from release, risk reduction, and controlled rollouts.
-   **What it sacrifices**: Adds a small amount of complexity to the code and requires managing environment variables.
-   **When to choose this approach**: For simple on/off toggles for new features, especially during development and initial launch phases.
-   **When to avoid this approach**: This simple pattern is not suitable for more complex scenarios like:
    -   **Percentage-based rollouts** (e.g., "show to 10% of users").
    -   **User-targeting** (e.g., "show only to users in Canada").
    -   **A/B testing**.
    For these, you should use a dedicated feature flagging service like LaunchDarkly, Vercel Flags, or Statsig.

**Limitation Preview**: Our app is now configurable and its features can be toggled. But we have no idea what users are doing. Are they even clicking "Add to Cart"? We are flying blind. Next, we need to add analytics.

## Analytics integration

## Iteration 2: Measuring User Behavior

Our `ProductDetailsPage` is deployed, configurable, and has a feature-flagged component. But from a business perspective, it's a black box. We need data to understand how users interact with our product.

**Current state recap**: We have a product page with an "Add to Cart" button.
**Current limitation**: We have no way of knowing if anyone ever clicks that button. We can't answer basic questions like "Which products are most popular?" or "Does the new reviews section increase cart additions?".

**New scenario introduction**: The marketing team wants to track every time a user clicks the "Add to Cart" button, along with the product ID and price.

### The Failure: Flying Blind

Without analytics, we are making business decisions based on guesswork. The "failure" is a complete lack of visibility into user behavior, which prevents us from improving our product.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**: A user clicks "Add to Cart". The UI might update, but no tracking information is sent.

**Network Tab Analysis**:
-   Request pattern: No requests are sent to any analytics service upon button click.
-   Evidence: The only network activity is the initial page load and data fetch.

**Let's parse this evidence**:

1.  **What the user experiences**: A normal button click.
2.  **What the business experiences**: Zero data.
3.  **Root cause identified**: The application lacks any client-side instrumentation to report user actions.
4.  **Why the current approach can't solve this**: Our component is a Server Component. User interactions like clicks happen in the browser, on the client. We need to introduce client-side interactivity and a way to communicate with a third-party analytics service.
5.  **What we need**:
    -   A way to handle client-side events (`onClick`). This requires converting part of our page to a Client Component.
    -   A small, reusable library for sending analytics events.
    -   Configuration for the analytics service (e.g., an tracking ID).

### Technique Introduced: Client Components and Analytics Abstraction

To handle user interactions, we must use a Client Component (`"use client"`). We'll extract the interactive parts of our page into a new component.

We will also create a simple analytics library (`lib/analytics.ts`) to abstract the details of the specific analytics provider we're using. This is a crucial pattern: it means if we ever switch from Google Analytics to Mixpanel, we only have to change this one file, not every single component that tracks an event.

### Solution Implementation

**Step 1: Create the Analytics Abstraction**

**Project Structure**:
```
src/
└── lib/
    └── analytics.ts   ← New analytics helper
```

```typescript
// src/lib/analytics.ts

// This is a mock analytics library. In a real app, you would install
// a package like `@vercel/analytics` or `analytics.js` and call its methods here.

// We declare the global `window.analytics` object for TypeScript
declare global {
  interface Window {
    // Replace with your analytics provider's actual interface
    analytics?: {
      track: (eventName: string, properties: Record<string, any>) => void;
    };
  }
}

export const trackEvent = (eventName: string, properties: Record<string, any>) => {
  // Check if the analytics script has loaded and the function is available
  if (window.analytics && typeof window.analytics.track === 'function') {
    window.analytics.track(eventName, properties);
  }

  // Also log to the console during development for easy debugging
  if (process.env.NODE_ENV === 'development') {
    console.log(`[Analytics Event]: ${eventName}`, properties);
  }
};
```

**Step 2: Add the Analytics Provider Script**

Most analytics providers require you to add a script tag to your site. We'll do this in the root `layout.tsx` file. We'll also configure it with an environment variable.

First, add the new variable to your `.env` files:
```bash
# .env.development.local
# ...
NEXT_PUBLIC_ANALYTICS_ID="G-DEV-12345"
```
```bash
# .env.production.local
# ...
NEXT_PUBLIC_ANALYTICS_ID="G-PROD-67890"
```

Now, update the root layout.

```tsx
// src/app/layout.tsx
import Script from 'next/script';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  const analyticsId = process.env.NEXT_PUBLIC_ANALYTICS_ID;

  return (
    <html lang="en">
      <body>{children}</body>
      {/* 
        In a real app, this would be the script from your provider.
        We'll use a mock script to simulate the `window.analytics` object.
      */}
      {analyticsId && (
        <Script id="mock-analytics" strategy="afterInteractive">
          {`
            window.analytics = {
              track: (name, props) => {
                console.log('Analytics call to provider:', name, props);
                // This is where the real provider would send a network request.
              }
            };
            console.log('Mock Analytics Initialized with ID: ${analyticsId}');
          `}
        </Script>
      )}
    </html>
  )
}
```

**Step 3: Refactor the Product Page into a Client Component**

Our `ProductDetailsPage` is a Server Component, which can't have interactive event handlers like `onClick`. We need to extract the part that needs to be interactive (the "Add to Cart" button and its logic) into a new Client Component.

**Project Structure**:
```
src/
├── app/
│   └── product/
│       └── [productId]/
│           └── page.tsx
└── components/
    ├── CustomerReviews.tsx
    └── ProductActions.tsx   ← New Client Component
```

**New Client Component**:

```tsx
// src/components/ProductActions.tsx
"use client"; // ← This marks it as a Client Component

import { trackEvent } from '@/lib/analytics';

type ProductActionsProps = {
  productId: string;
  productName: string;
  price: number;
};

export default function ProductActions({ productId, productName, price }: ProductActionsProps) {
  
  const handleAddToCart = () => {
    console.log('Add to cart clicked!');
    trackEvent('add_to_cart', {
      productId,
      productName,
      price,
      currency: 'USD',
    });
    // In a real app, you'd also add logic here to update a shopping cart context.
  };

  return (
    <button onClick={handleAddToCart}>Add to Cart</button>
  );
}
```

**Update the Main Page Component**:

Now we replace the static `<button>` in our Server Component with our new interactive `ProductActions` Client Component.

**Before**:
```tsx
// src/app/product/[productId]/page.tsx (snippet)
export default async function ProductDetailsPage({ params }) {
  // ...
  return (
    <main>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <strong>${product.price.toFixed(2)}</strong>
      <button>Add to Cart</button> {/* ← Static button */}
      {/* ... */}
    </main>
  );
}
```

**After**:
```tsx
// src/app/product/[productId]/page.tsx (Updated)
// ... (imports and getProduct are the same)
import ProductActions from '@/components/ProductActions'; // ← Import new component

export default async function ProductDetailsPage({ params }: { params: { productId: string } }) {
  const product = await getProduct(params.productId);
  const reviewsEnabled = process.env.NEXT_PUBLIC_FEATURE_REVIEWS_ENABLED === 'true';

  if (!product) {
    notFound();
  }

  return (
    <main>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <strong>${product.price.toFixed(2)}</strong>
      
      {/* ✅ Replace static button with interactive Client Component */}
      <ProductActions 
        productId={product.id} 
        productName={product.name} 
        price={product.price} 
      />

      {reviewsEnabled && <CustomerReviews productId={params.productId} />}
    </main>
  );
}
```

### Verification

1.  Run the app with `npm run dev`.
2.  Open the browser's developer tools and go to the Console tab.
3.  Navigate to a product page. You should see the message "Mock Analytics Initialized...".
4.  Click the "Add to Cart" button.

**Browser Console Output**:
```
Add to cart clicked!
[Analytics Event]: add_to_cart {productId: '123', productName: 'Super Widget', price: 99.99, currency: 'USD'}
Analytics call to provider: add_to_cart {productId: '123', productName: 'Super Widget', price: 99.99, currency: 'USD'}
```

This confirms that our `onClick` handler is firing, our `trackEvent` abstraction is being called, and it's successfully invoking the (mocked) third-party analytics library. In a real scenario, the Network tab would show a request being sent to the analytics provider's endpoint.

**Limitation Preview**: We can now track what users do successfully. But what happens when things go wrong? If our API fails or a client-side bug occurs, the user gets a broken experience, and we might never know about it. We need to add error monitoring.

## Monitoring and alerting

## Iteration 3: Detecting and Reporting Errors

Our application is now instrumented for success metrics (analytics), but not for failure metrics. When something breaks, we are still in the dark. We need a system to capture errors, report them to a centralized service, and alert the development team.

**Current state recap**: We can track user actions like adding an item to a cart.
**Current limitation**: If the `getProduct` fetch call fails, the user sees a generic `not-found` or `error` page, and our team is not notified. We only find out when a customer complains.

**New scenario introduction**: The product API has a bug and starts returning `500 Internal Server Error` for some product IDs. We need to be alerted automatically the moment this happens.

### The Failure: Silent Errors

Let's simulate the API failure. We'll modify `getProduct` to throw an error.

```typescript
// src/app/product/[productId]/page.tsx (modified for demonstration)

async function getProduct(productId: string): Promise<Product | null> {
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;
  
  // Simulate a 50% failure rate
  if (Math.random() > 0.5) {
    throw new Error("API Server Error: Failed to fetch product data.");
  }

  const res = await fetch(`${apiUrl}/products/${productId}`);
  
  if (!res.ok) {
    throw new Error("API Network Error: Invalid response.");
  }
  return res.json();
}
```
When this error is thrown in a Server Component during rendering, Next.js will catch it and render the nearest `error.js` boundary. If you don't have one, it provides a default error page in production.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**: The user sees a generic error page. For example, Next.js's default is "Application error: a server-side exception has occurred (see the server logs for more information)." It's unhelpful for the user and provides no immediate feedback to us.

**Terminal Output (Server Logs)**:
```bash
- error src/app/product/[productId]/page.tsx (15:11) @ getProduct
- error Error: API Server Error: Failed to fetch product data.
    at getProduct (./src/app/product/[productId]/page.tsx:21:11)
    at ProductDetailsPage (./src/app/product/[productId]/page.tsx:32:21)
    ...
```

**Let's parse this evidence**:

1.  **What the user experiences**: A broken page with a generic, unhelpful message.
2.  **What the developer experiences**: The error is logged to the server console, but if you're running on a platform like Vercel, you have to manually go digging through logs to find it. There are no automatic alerts. For a client-side error, it would only appear in the user's browser console, which is completely invisible to us.
3.  **Root cause identified**: The application has no mechanism to report exceptions to a dedicated monitoring service.
4.  **Why the current approach can't solve this**: `console.error` is not a monitoring strategy. Logs are passive; we need an active system that aggregates errors and pushes alerts.
5.  **What we need**: An error monitoring service (like Sentry, LogRocket, or Datadog) and a way to report errors to it from both the server and the client.

### Technique Introduced: Centralized Error Reporting

We'll follow a similar pattern to our analytics library by creating a simple `lib/monitoring.ts` abstraction. This will allow us to capture errors and send them to a third-party service.

### Solution Implementation

**Step 1: Create the Monitoring Abstraction**

**Project Structure**:
```
src/
└── lib/
    ├── analytics.ts
    └── monitoring.ts   ← New monitoring helper
```

```typescript
// src/lib/monitoring.ts

// This is a mock monitoring library. In a real app, you would install
// a package like `@sentry/nextjs` and call its methods here.

type UserInfo = {
  id?: string;
  email?: string;
};

export const reportError = (error: Error, context: Record<string, any> = {}, user?: UserInfo) => {
  // In a real app, this would send the error to Sentry, Datadog, etc.
  // This service would then aggregate errors and send alerts (e.g., to Slack).
  
  const errorReport = {
    message: error.message,
    stack: error.stack,
    context,
    user,
    timestamp: new Date().toISOString(),
  };

  console.error("[Error Report Sent]:", JSON.stringify(errorReport, null, 2));

  // Here you would call the actual SDK, e.g., Sentry.captureException(error);
};
```

**Step 2: Integrate Error Reporting into Data Fetching**

Now, we'll wrap our `getProduct` function's logic in a `try...catch` block to report any failures.

**Before**:
```typescript
// src/app/product/[productId]/page.tsx (snippet)
async function getProduct(productId: string): Promise<Product | null> {
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;
  const res = await fetch(`${apiUrl}/products/${productId}`);
  
  if (!res.ok) {
    // This just renders a not-found page, we don't know it happened.
    return null;
  }
  return res.json();
}
```

**After**:
```typescript
// src/app/product/[productId]/page.tsx (snippet)
import { reportError } from '@/lib/monitoring'; // ← Import

async function getProduct(productId: string): Promise<Product | null> {
  try {
    const apiUrl = process.env.NEXT_PUBLIC_API_URL;
    const res = await fetch(`${apiUrl}/products/${productId}`);
    
    if (!res.ok) {
      // Create a more informative error
      const error = new Error(`API request failed with status ${res.status}`);
      // Throw it so Next.js can catch it and render the error boundary
      throw error;
    }
    return res.json();
  } catch (error) {
    if (error instanceof Error) {
      // ✅ Report the error to our monitoring service
      reportError(error, {
        productId,
        component: 'ProductDetailsPage',
        source: 'getProduct',
      });
    }
    // Re-throw the error so that Next.js's error handling still takes over
    throw error;
  }
}
```
*Note: We re-throw the error so that Next.js still renders the `error.js` boundary. Our `reportError` function is a side-effect.*

**Step 3: Create a Custom Error UI**

To give the user a better experience, we can create a custom `error.js` file. This file defines a UI boundary for runtime errors.

**Project Structure**:
```
src/
└── app/
    └── product/
        └── [productId]/
            ├── page.tsx
            └── error.tsx   ← New error boundary UI
```

```tsx
// src/app/product/[productId]/error.tsx
"use client"; // Error components must be Client Components

import { useEffect } from 'react';
import { reportError } from '@/lib/monitoring';

export default function ProductError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Report the error to our monitoring service
    // This captures errors that happen during rendering on the client
    reportError(error, { component: "ProductErrorUI" });
  }, [error]);

  return (
    <div>
      <h2>Something went wrong!</h2>
      <p>We're sorry, we couldn't load the product details. Please try again later.</p>
      <button onClick={() => reset()}>
        Try again
      </button>
    </div>
  );
}
```

### Verification

1.  Re-introduce the simulated error in `getProduct`.
2.  Run the app and navigate to a product page.
3.  The request will eventually fail.

**Browser Behavior**: Instead of the default Next.js error page, the user now sees our custom UI from `error.tsx` with a "Something went wrong!" message and a "Try again" button.

**Terminal Output (Server Logs)**:
You will see our detailed error report logged to the console, simulating it being sent to a service like Sentry.

```bash
[Error Report Sent]: {
  "message": "API request failed with status 500",
  "stack": "Error: API request failed with status 500\n    at getProduct (...)",
  "context": {
    "productId": "123",
    "component": "ProductDetailsPage",
    "source": "getProduct"
  },
  "timestamp": "2023-10-27T10:00:00.000Z"
}
- error Error: API request failed with status 500
... (Next.js default error log)
```

This structured log is what a monitoring service would use to create an issue, group similar errors, and send an alert to your team's Slack channel, giving you immediate visibility into production problems.

## The deployment checklist

## Synthesis: The Pre-Flight Checklist

We have taken our simple `ProductDetailsPage` and progressively hardened it for production. We've made it configurable, added feature flags, integrated analytics, and set up error monitoring. These are the core pillars of a production-ready application.

This final section synthesizes these concepts into a practical checklist to review before every deployment.

### The Journey: From Problem to Solution

| Iteration | Failure Mode                               | Technique Applied                     | Result                                                              |
| :-------- | :----------------------------------------- | :------------------------------------ | :------------------------------------------------------------------ |
| 0         | Hardcoded API URLs, not portable.          | None (Initial State)                  | Works only in one environment.                                      |
| 1         | All-or-nothing feature releases.           | Environment Variables for Feature Flags | Features can be enabled/disabled at runtime without a new deploy.   |
| 2         | No visibility into user behavior.          | Analytics Abstraction & Client Component | User interactions are tracked, providing valuable product data.     |
| 3         | Silent failures, bugs go unnoticed.        | Centralized Error Reporting & `error.js` | Errors are captured, reported, and users see a friendly UI.         |

### Final Implementation

Here is the complete, production-ready version of our `page.tsx` and its supporting components, incorporating all the improvements.

<code language="tsx">
// src/app/product/[productId]/page.tsx (Final Version)

import { notFound } from 'next/navigation';
import { reportError } from '@/lib/monitoring';
import CustomerReviews from '@/components/CustomerReviews';
import ProductActions from '@/components/ProductActions';

type Product = {
  id: string;
  name:string;
  price: number;
  description: string;
};

async function getProduct(productId: string): Promise<Product | null> {
  try {
    const apiUrl = process.env.NEXT_PUBLIC_API_URL;
    if (!apiUrl) {
      throw new Error("Configuration Error: NEXT_PUBLIC_API_URL is not defined.");
    }
    const res = await fetch(`${apiUrl}/products/${productId}`, { next: { revalidate: 3600 } }); // Add revalidation
    
    if (!res.ok) {
      throw new Error(`API request failed with status ${res.status}`);
    }
    return res.json();
  } catch (error) {
    if (error instanceof Error) {
      reportError(error, { productId, component: 'ProductDetailsPage', source: 'getProduct' });
    }
    // Let Next.js handle rendering the error boundary
    throw error;
  }
}

export default async function ProductDetailsPage({ params }: { params: { productId: string } }) {
  // The `getProduct` call is wrapped in Next.js's error handling
  const product = await getProduct(params.productId);
  
  const reviewsEnabled = process.env.NEXT_PUBLIC_FEATURE_REVIEWS_ENABLED === 'true';

  if (!product) {
    notFound();
  }

  return (
    <main>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <strong>${product.price.toFixed(2)}</strong>
      
      <ProductActions 
        productId={product.id} 
        productName={product.name} 
        price={product.price} 
      />

      {reviewsEnabled && <CustomerReviews productId={params.productId} />}
    </main>
  );
}
    ```
    
### The Deployment Checklist

Use this as a final check before merging to your main branch and deploying.

**✅ Configuration & Security**
- [ ] All secret keys (database URLs, private API keys) are stored in environment variables, NOT committed to Git.
- [ ] The `.env.local` and other sensitive `.env` files are in `.gitignore`.
- [ ] An `.env.example` file exists to show other developers what variables are needed.
- [ ] Environment variables are correctly prefixed (`NEXT_PUBLIC_` for browser access, no prefix for server-only).
- [ ] Run `npm audit` to check for known vulnerabilities in dependencies.

**✅ Code Quality & Build**
- [ ] The code passes all linter (`next lint`) and formatter (e.g., Prettier) checks.
- [ ] The TypeScript compiler runs without errors (`npx tsc --noEmit`).
- [ ] The production build completes successfully (`npm run build`).
- [ ] Review the build output for unexpectedly large page sizes or chunks.
- [ ] All `console.log` statements used for debugging have been removed.

**✅ Functionality & Testing**
- [ ] All new features are controlled by feature flags, set to `false` in production by default.
- [ ] Unit and integration tests pass in your CI/CD pipeline.
- [ ] End-to-end tests (e.g., with Cypress or Playwright) pass for critical user flows.
- [ ] The application has been tested on major browsers (Chrome, Firefox, Safari).
- [ ] The application is responsive and works on mobile devices.

**✅ Analytics & Monitoring**
- [ ] Analytics events are implemented for all key user actions.
- [ ] Error reporting is configured for both server-side and client-side exceptions.
- [ ] Custom `error.tsx` and `not-found.tsx` pages are in place to provide a good user experience on failure.
- [ ] Alerts are configured in your monitoring service to notify the team of critical errors.

**✅ Performance**
- [ ] Run a Lighthouse audit on key pages. Aim for scores above 90.
- [ ] Check Core Web Vitals (LCP, FID/INP, CLS).
- [ ] Analyze the bundle with `@next/bundle-analyzer` to identify and remove unnecessary dependencies.
- [ ] Ensure images are optimized using `next/image`.
- [ ] Caching strategies (e.g., `revalidate` times, `cache` function) are set appropriately for data fetches.

**✅ Deployment & Rollback**
- [ ] The deployment process is automated via a CI/CD pipeline.
- [ ] You have a clear and tested plan for rolling back to the previous version if the deployment fails.
- [ ] The team is aware of the deployment and is ready to monitor the release.

Going to production is more than just running a command. It's a disciplined process of verification that ensures your application is robust, secure, and observable. By following these steps, you can deploy with confidence.
