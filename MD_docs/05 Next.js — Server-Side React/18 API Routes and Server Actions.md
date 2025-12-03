# Chapter 18: API Routes and Server Actions

## Building API endpoints

## The Problem: Client-Side Form Submission Limitations

Before we dive into API routes, let's understand why we need them. In Chapter 17, we built our e-commerce product catalog with Server Components fetching data. But what happens when users need to **modify** data—adding products to a cart, submitting reviews, or updating their profile?

Let's start with a naive approach: handling everything client-side.

### Reference Implementation: Product Review System

We'll build a product review submission system that evolves through this chapter. Users can submit reviews with ratings and text. This seemingly simple feature will expose multiple failure modes that drive us toward better solutions.

**Project Structure**:
```
src/
├── app/
│   ├── products/
│   │   └── [id]/
│   │       ├── page.tsx          ← Product detail page
│   │       └── ReviewForm.tsx    ← Our reference implementation
│   └── api/
│       └── reviews/
│           └── route.ts          ← API endpoint (we'll build this)
└── lib/
    └── db.ts                     ← Database utilities
```

Here's our initial client-side approach:

```tsx
// src/app/products/[id]/ReviewForm.tsx
'use client';

import { useState } from 'react';

export function ReviewForm({ productId }: { productId: string }) {
  const [rating, setRating] = useState(5);
  const [comment, setComment] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setIsSubmitting(true);

    try {
      // Directly calling an external API from the client
      const response = await fetch('https://api.example.com/reviews', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          productId,
          rating,
          comment,
          apiKey: 'sk_live_abc123xyz', // 🚨 EXPOSED SECRET!
        }),
      });

      if (!response.ok) {
        throw new Error('Failed to submit review');
      }

      alert('Review submitted!');
      setRating(5);
      setComment('');
    } catch (error) {
      alert('Error submitting review');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label>Rating:</label>
        <select value={rating} onChange={(e) => setRating(Number(e.target.value))}>
          {[1, 2, 3, 4, 5].map((n) => (
            <option key={n} value={n}>{n} stars</option>
          ))}
        </select>
      </div>
      
      <div>
        <label>Comment:</label>
        <textarea
          value={comment}
          onChange={(e) => setComment(e.target.value)}
          rows={4}
        />
      </div>
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit Review'}
      </button>
    </form>
  );
}
```

### The Failure: Security and Architecture Problems

Let's run this code and examine what happens.

**Browser Behavior**:
The form appears to work—users can submit reviews, and they see a success message. But open the browser's DevTools Network tab.

**Browser DevTools - Network Tab**:
- Filter: Fetch/XHR
- Request to: `https://api.example.com/reviews`
- Request Headers visible in DevTools
- Request Payload visible: `{ "productId": "123", "rating": 5, "comment": "Great!", "apiKey": "sk_live_abc123xyz" }`

**Security Console (Hypothetical)**:
```
[SECURITY ALERT] API key 'sk_live_abc123xyz' exposed in client-side code
[SECURITY ALERT] 47 unauthorized requests detected using leaked key
[SECURITY ALERT] $2,847 in fraudulent charges
```

**Let's parse this evidence**:

1. **What the user experiences**: The form works perfectly from their perspective.

2. **What DevTools reveals**: Every piece of data sent to the server is visible in the Network tab, including the API key embedded in the request body.

3. **What actually happened**: 
   - The API key is bundled into the client JavaScript
   - Anyone can view the source code and extract it
   - Malicious actors can use the key to make unlimited requests
   - There's no rate limiting or authentication

4. **Root cause identified**: **Secrets cannot live in client-side code**. Anything sent to the browser is public information.

5. **Why the current approach can't solve this**: Client-side JavaScript is inherently public. No amount of obfuscation or "hiding" can protect secrets in client code.

6. **What we need**: A server-side layer that holds secrets and validates requests before forwarding them to external services.

### The Solution: API Routes

Next.js API Routes provide server-side endpoints that run in a Node.js environment. They can:
- Hold secrets securely
- Validate and sanitize input
- Authenticate users
- Rate limit requests
- Transform data before sending to external services

Let's build our first API route.

```typescript
// src/app/api/reviews/route.ts
import { NextRequest, NextResponse } from 'next/server';

// This runs on the server - secrets are safe here
const API_KEY = process.env.EXTERNAL_API_KEY!;

export async function POST(request: NextRequest) {
  try {
    // Parse the incoming request body
    const body = await request.json();
    const { productId, rating, comment } = body;

    // Server-side validation
    if (!productId || !rating || !comment) {
      return NextResponse.json(
        { error: 'Missing required fields' },
        { status: 400 }
      );
    }

    if (rating < 1 || rating > 5) {
      return NextResponse.json(
        { error: 'Rating must be between 1 and 5' },
        { status: 400 }
      );
    }

    // Call external API with secret key (never exposed to client)
    const response = await fetch('https://api.example.com/reviews', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${API_KEY}`, // Secret stays on server
      },
      body: JSON.stringify({
        productId,
        rating,
        comment,
        timestamp: new Date().toISOString(),
      }),
    });

    if (!response.ok) {
      throw new Error('External API request failed');
    }

    const data = await response.json();

    return NextResponse.json(
      { success: true, reviewId: data.id },
      { status: 201 }
    );
  } catch (error) {
    console.error('Review submission error:', error);
    return NextResponse.json(
      { error: 'Failed to submit review' },
      { status: 500 }
    );
  }
}
```

Now update the client component to use our API route:

```tsx
// src/app/products/[id]/ReviewForm.tsx
'use client';

import { useState } from 'react';

export function ReviewForm({ productId }: { productId: string }) {
  const [rating, setRating] = useState(5);
  const [comment, setComment] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setIsSubmitting(true);
    setError(null);

    try {
      // Call OUR API route, not external API directly
      const response = await fetch('/api/reviews', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          productId,
          rating,
          comment,
          // No API key needed - server handles it
        }),
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(data.error || 'Failed to submit review');
      }

      alert('Review submitted successfully!');
      setRating(5);
      setComment('');
    } catch (error) {
      setError(error instanceof Error ? error.message : 'An error occurred');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {error && (
        <div style={{ color: 'red', marginBottom: '1rem' }}>
          {error}
        </div>
      )}
      
      <div>
        <label>Rating:</label>
        <select value={rating} onChange={(e) => setRating(Number(e.target.value))}>
          {[1, 2, 3, 4, 5].map((n) => (
            <option key={n} value={n}>{n} stars</option>
          ))}
        </select>
      </div>
      
      <div>
        <label>Comment:</label>
        <textarea
          value={comment}
          onChange={(e) => setComment(e.target.value)}
          rows={4}
          required
        />
      </div>
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit Review'}
      </button>
    </form>
  );
}
```

**Environment Setup**:

```bash
# .env.local (never commit this file!)
EXTERNAL_API_KEY=sk_live_abc123xyz
```

### Verification: Security Restored

**Browser DevTools - Network Tab**:
- Request to: `/api/reviews` (our domain, not external API)
- Request Payload: `{ "productId": "123", "rating": 5, "comment": "Great!" }`
- **No API key visible anywhere**

**Browser DevTools - Sources Tab**:
- Search for "sk_live" in all JavaScript files
- **Result**: Not found (key never sent to client)

**Server Terminal Output**:
```
POST /api/reviews 201 in 234ms
```

**Expected vs. Actual Improvement**:
- **Before**: API key exposed in client bundle, visible in Network tab
- **After**: API key stays on server, never sent to client
- **Security**: Secrets protected, rate limiting possible, validation enforced

### API Route Anatomy

Let's break down the structure:

```typescript
// src/app/api/reviews/route.ts

// 1. Import Next.js types
import { NextRequest, NextResponse } from 'next/server';

// 2. Export named functions for HTTP methods
export async function GET(request: NextRequest) {
  // Handle GET requests
}

export async function POST(request: NextRequest) {
  // Handle POST requests
}

export async function PUT(request: NextRequest) {
  // Handle PUT requests
}

export async function DELETE(request: NextRequest) {
  // Handle DELETE requests
}

// 3. Access request data
export async function POST(request: NextRequest) {
  // URL parameters
  const { searchParams } = new URL(request.url);
  const id = searchParams.get('id');

  // Request body
  const body = await request.json();

  // Headers
  const authHeader = request.headers.get('authorization');

  // Cookies
  const token = request.cookies.get('token');

  // Return response
  return NextResponse.json({ data: 'value' }, { status: 200 });
}
```

### Dynamic Route Parameters

API routes support dynamic segments just like page routes:

```typescript
// src/app/api/reviews/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

// GET /api/reviews/123
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const reviewId = params.id;

  // Fetch specific review
  const review = await fetchReviewById(reviewId);

  if (!review) {
    return NextResponse.json(
      { error: 'Review not found' },
      { status: 404 }
    );
  }

  return NextResponse.json(review);
}

// DELETE /api/reviews/123
export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const reviewId = params.id;

  // Verify user owns this review (authentication check)
  const userId = await getUserIdFromRequest(request);
  const review = await fetchReviewById(reviewId);

  if (review.userId !== userId) {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 403 }
    );
  }

  await deleteReview(reviewId);

  return NextResponse.json({ success: true });
}

// Helper functions (implementation details)
async function fetchReviewById(id: string) {
  // Database query
  return { id, userId: 'user123', rating: 5, comment: 'Great!' };
}

async function getUserIdFromRequest(request: NextRequest) {
  // Extract from session/JWT
  return 'user123';
}

async function deleteReview(id: string) {
  // Database deletion
}
```

### Iteration 1: Adding Database Integration

Our API route currently calls an external API. In most real applications, you'll store data in your own database. Let's add that.

**Current limitation**: We're proxying to an external service, adding latency and dependency on third-party availability.

**New scenario**: What if we want to store reviews in our own database for faster access and better control?

```typescript
// src/lib/db.ts
// Simple in-memory database for demonstration
// In production, use Prisma, Drizzle, or your preferred ORM

type Review = {
  id: string;
  productId: string;
  userId: string;
  rating: number;
  comment: string;
  createdAt: Date;
};

const reviews: Review[] = [];

export const db = {
  reviews: {
    create: async (data: Omit<Review, 'id' | 'createdAt'>) => {
      const review: Review = {
        ...data,
        id: Math.random().toString(36).substring(7),
        createdAt: new Date(),
      };
      reviews.push(review);
      return review;
    },

    findByProductId: async (productId: string) => {
      return reviews.filter((r) => r.productId === productId);
    },

    findById: async (id: string) => {
      return reviews.find((r) => r.id === id);
    },

    delete: async (id: string) => {
      const index = reviews.findIndex((r) => r.id === id);
      if (index > -1) {
        reviews.splice(index, 1);
        return true;
      }
      return false;
    },
  },
};
```

Now update the API route to use our database:

```typescript
// src/app/api/reviews/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { db } from '@/lib/db';

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { productId, rating, comment } = body;

    // Validation
    if (!productId || !rating || !comment) {
      return NextResponse.json(
        { error: 'Missing required fields' },
        { status: 400 }
      );
    }

    if (rating < 1 || rating > 5) {
      return NextResponse.json(
        { error: 'Rating must be between 1 and 5' },
        { status: 400 }
      );
    }

    if (comment.length < 10) {
      return NextResponse.json(
        { error: 'Comment must be at least 10 characters' },
        { status: 400 }
      );
    }

    // Get user ID from session (we'll implement auth in Chapter 20)
    // For now, use a placeholder
    const userId = 'user123';

    // Store in database
    const review = await db.reviews.create({
      productId,
      userId,
      rating,
      comment,
    });

    return NextResponse.json(
      { success: true, review },
      { status: 201 }
    );
  } catch (error) {
    console.error('Review submission error:', error);
    return NextResponse.json(
      { error: 'Failed to submit review' },
      { status: 500 }
    );
  }
}

// GET /api/reviews?productId=123
export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const productId = searchParams.get('productId');

  if (!productId) {
    return NextResponse.json(
      { error: 'productId is required' },
      { status: 400 }
    );
  }

  const reviews = await db.reviews.findByProductId(productId);

  return NextResponse.json({ reviews });
}
```

### Verification: Database Integration Working

**Browser DevTools - Network Tab**:
```
POST /api/reviews
Status: 201 Created
Response: {
  "success": true,
  "review": {
    "id": "a7b3c9d",
    "productId": "123",
    "userId": "user123",
    "rating": 5,
    "comment": "Excellent product!",
    "createdAt": "2025-01-15T10:30:00.000Z"
  }
}
```

**Server Terminal Output**:
```
POST /api/reviews 201 in 12ms
```

**Expected vs. Actual Improvement**:
- **Before**: Proxying to external API (200-500ms latency)
- **After**: Direct database access (10-20ms latency)
- **Performance**: 10-20x faster response time
- **Control**: Full ownership of data, no third-party dependency

### Limitation Preview

This works well, but we still have a problem: **the client must handle all the submission logic**. If JavaScript fails to load or is disabled, the form doesn't work at all. We also need to manually manage loading states, error states, and success states.

In the next section, we'll see how Server Actions eliminate this boilerplate while providing progressive enhancement.

## Server Actions: mutations without API routes

## The Problem: API Routes Require Client-Side Orchestration

Our API route works, but look at all the client-side code required:

1. Event handler to prevent default form submission
2. Manual state management for loading/error states
3. Fetch call with proper headers and error handling
4. Response parsing and validation
5. UI updates based on response

**The failure**: If JavaScript fails to load (slow network, JS disabled, error in bundle), the form is completely non-functional. Users see a form but can't submit it.

**Diagnostic Analysis: Simulating JavaScript Failure**:

**Browser DevTools - Network Tab**:
- Throttle to "Slow 3G"
- Disable JavaScript in DevTools Settings
- Try to submit form

**Browser Behavior**:
- Form appears normal
- Click submit button
- **Nothing happens**
- No feedback, no error message
- Form is a dead UI element

**Console Output**:
```
(No output - JavaScript never executed)
```

**Root cause identified**: The form depends entirely on JavaScript for functionality. Without JS, it's just HTML with no behavior.

**What we need**: A way to handle form submissions that works with or without JavaScript, while still providing enhanced UX when JS is available.

### Server Actions: The Modern Solution

Server Actions are functions that run on the server but can be called directly from Client Components or Server Components. They provide:

1. **Progressive enhancement**: Forms work without JavaScript
2. **Type safety**: Full TypeScript support from client to server
3. **Automatic serialization**: No manual JSON parsing
4. **Built-in error handling**: Structured error responses
5. **Optimistic updates**: Easy to implement

Let's refactor our review form to use Server Actions.

```typescript
// src/app/actions/reviews.ts
'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

// Server Action - runs on server, callable from client
export async function submitReview(formData: FormData) {
  // Extract form data
  const productId = formData.get('productId') as string;
  const rating = Number(formData.get('rating'));
  const comment = formData.get('comment') as string;

  // Validation
  if (!productId || !rating || !comment) {
    return {
      success: false,
      error: 'Missing required fields',
    };
  }

  if (rating < 1 || rating > 5) {
    return {
      success: false,
      error: 'Rating must be between 1 and 5',
    };
  }

  if (comment.length < 10) {
    return {
      success: false,
      error: 'Comment must be at least 10 characters',
    };
  }

  try {
    // Get user ID from session (placeholder for now)
    const userId = 'user123';

    // Store in database
    const review = await db.reviews.create({
      productId,
      userId,
      rating,
      comment,
    });

    // Revalidate the product page to show new review
    revalidatePath(`/products/${productId}`);

    return {
      success: true,
      review,
    };
  } catch (error) {
    console.error('Review submission error:', error);
    return {
      success: false,
      error: 'Failed to submit review',
    };
  }
}
```

Now update the form component to use the Server Action:

```tsx
// src/app/products/[id]/ReviewForm.tsx
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { submitReview } from '@/app/actions/reviews';

// Separate component for submit button to access form status
function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit Review'}
    </button>
  );
}

export function ReviewForm({ productId }: { productId: string }) {
  const [state, formAction] = useFormState(submitReview, {
    success: false,
    error: null,
  });

  return (
    <form action={formAction}>
      {/* Hidden field for productId */}
      <input type="hidden" name="productId" value={productId} />

      {state.error && (
        <div style={{ color: 'red', marginBottom: '1rem' }}>
          {state.error}
        </div>
      )}

      {state.success && (
        <div style={{ color: 'green', marginBottom: '1rem' }}>
          Review submitted successfully!
        </div>
      )}
      
      <div>
        <label htmlFor="rating">Rating:</label>
        <select id="rating" name="rating" defaultValue="5" required>
          {[1, 2, 3, 4, 5].map((n) => (
            <option key={n} value={n}>{n} stars</option>
          ))}
        </select>
      </div>
      
      <div>
        <label htmlFor="comment">Comment:</label>
        <textarea
          id="comment"
          name="comment"
          rows={4}
          required
          minLength={10}
        />
      </div>
      
      <SubmitButton />
    </form>
  );
}
```

### Verification: Progressive Enhancement Working

**Test 1: With JavaScript Enabled**

**Browser Behavior**:
- Fill out form
- Click submit
- Button shows "Submitting..." immediately
- Success message appears
- Form stays on same page (no full reload)

**Browser DevTools - Network Tab**:
```
POST /products/123
Status: 200 OK
Type: document (Server Action request)
```

**React DevTools - Components Tab**:
- `ReviewForm` component selected
- State: `{ success: true, error: null }`
- No full page reload occurred

**Test 2: With JavaScript Disabled**

**Browser DevTools**:
- Settings → Disable JavaScript
- Refresh page

**Browser Behavior**:
- Fill out form
- Click submit
- **Page reloads** (full navigation)
- Success message appears on reloaded page
- Form still works!

**Server Terminal Output**:
```
POST /products/123 (Server Action: submitReview)
Review created: a7b3c9d
Revalidating path: /products/123
200 OK in 15ms
```

**Expected vs. Actual Improvement**:
- **Before**: Form completely broken without JavaScript
- **After**: Form works with or without JavaScript
- **With JS**: Enhanced UX (no page reload, instant feedback)
- **Without JS**: Graceful degradation (full page reload, still functional)
- **Code reduction**: ~40 lines of client code → ~20 lines

### How Server Actions Work

Let's understand the mechanism:

1. **With JavaScript**: 
   - Form submission intercepted by React
   - Server Action called via fetch (automatic)
   - Response updates component state
   - No page reload

2. **Without JavaScript**:
   - Form submits as standard HTML form
   - Browser performs full POST request
   - Server processes action
   - Server returns new HTML
   - Page reloads with result

The same server function handles both cases!

### Iteration 2: Type-Safe Server Actions

The FormData approach works but lacks type safety. Let's add proper TypeScript types.

```typescript
// src/app/actions/reviews.ts
'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';
import { z } from 'zod';

// Define validation schema
const reviewSchema = z.object({
  productId: z.string().min(1, 'Product ID is required'),
  rating: z.number().min(1).max(5, 'Rating must be between 1 and 5'),
  comment: z.string().min(10, 'Comment must be at least 10 characters'),
});

// Type-safe return type
type ReviewActionResult = 
  | { success: true; review: { id: string; rating: number; comment: string } }
  | { success: false; error: string; fieldErrors?: Record<string, string[]> };

export async function submitReview(
  prevState: ReviewActionResult | null,
  formData: FormData
): Promise<ReviewActionResult> {
  // Parse and validate form data
  const rawData = {
    productId: formData.get('productId') as string,
    rating: Number(formData.get('rating')),
    comment: formData.get('comment') as string,
  };

  const validation = reviewSchema.safeParse(rawData);

  if (!validation.success) {
    return {
      success: false,
      error: 'Validation failed',
      fieldErrors: validation.error.flatten().fieldErrors,
    };
  }

  const { productId, rating, comment } = validation.data;

  try {
    const userId = 'user123'; // Placeholder

    const review = await db.reviews.create({
      productId,
      userId,
      rating,
      comment,
    });

    revalidatePath(`/products/${productId}`);

    return {
      success: true,
      review: {
        id: review.id,
        rating: review.rating,
        comment: review.comment,
      },
    };
  } catch (error) {
    console.error('Review submission error:', error);
    return {
      success: false,
      error: 'Failed to submit review. Please try again.',
    };
  }
}
```

Update the form to display field-specific errors:

```tsx
// src/app/products/[id]/ReviewForm.tsx
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { submitReview } from '@/app/actions/reviews';

function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit Review'}
    </button>
  );
}

export function ReviewForm({ productId }: { productId: string }) {
  const [state, formAction] = useFormState(submitReview, null);

  return (
    <form action={formAction}>
      <input type="hidden" name="productId" value={productId} />

      {state?.error && !state.fieldErrors && (
        <div style={{ color: 'red', marginBottom: '1rem' }}>
          {state.error}
        </div>
      )}

      {state?.success && (
        <div style={{ color: 'green', marginBottom: '1rem' }}>
          Review submitted successfully!
        </div>
      )}
      
      <div>
        <label htmlFor="rating">Rating:</label>
        <select id="rating" name="rating" defaultValue="5" required>
          {[1, 2, 3, 4, 5].map((n) => (
            <option key={n} value={n}>{n} stars</option>
          ))}
        </select>
        {state?.fieldErrors?.rating && (
          <p style={{ color: 'red', fontSize: '0.875rem' }}>
            {state.fieldErrors.rating[0]}
          </p>
        )}
      </div>
      
      <div>
        <label htmlFor="comment">Comment:</label>
        <textarea
          id="comment"
          name="comment"
          rows={4}
          required
          minLength={10}
        />
        {state?.fieldErrors?.comment && (
          <p style={{ color: 'red', fontSize: '0.875rem' }}>
            {state.fieldErrors.comment[0]}
          </p>
        )}
      </div>
      
      <SubmitButton />
    </form>
  );
}
```

### Verification: Type-Safe Validation

**Test: Submit Invalid Data**

**Browser Behavior**:
- Enter rating: 5
- Enter comment: "Bad" (too short)
- Click submit

**Browser Console Output**:
```
(No errors - validation handled server-side)
```

**UI Display**:
```
Comment must be at least 10 characters
```

**Server Terminal Output**:
```
POST /products/123 (Server Action: submitReview)
Validation failed: comment too short
200 OK in 3ms
```

**Expected vs. Actual Improvement**:
- **Before**: Generic error messages, no field-specific feedback
- **After**: Precise error messages per field
- **Type safety**: Full TypeScript inference from server to client
- **Validation**: Centralized on server, can't be bypassed

### Iteration 3: Optimistic Updates

Server Actions make optimistic updates trivial. Let's show the review immediately while the server processes it.

```tsx
// src/app/products/[id]/ReviewForm.tsx
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { useOptimistic } from 'react';
import { submitReview } from '@/app/actions/reviews';

type Review = {
  id: string;
  rating: number;
  comment: string;
  pending?: boolean;
};

function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit Review'}
    </button>
  );
}

export function ReviewForm({ 
  productId,
  existingReviews = [],
}: { 
  productId: string;
  existingReviews?: Review[];
}) {
  const [state, formAction] = useFormState(submitReview, null);
  const [optimisticReviews, addOptimisticReview] = useOptimistic(
    existingReviews,
    (state, newReview: Review) => [...state, newReview]
  );

  async function handleSubmit(formData: FormData) {
    // Add optimistic review immediately
    const rating = Number(formData.get('rating'));
    const comment = formData.get('comment') as string;
    
    addOptimisticReview({
      id: 'temp-' + Date.now(),
      rating,
      comment,
      pending: true,
    });

    // Submit to server
    await formAction(formData);
  }

  return (
    <div>
      {/* Display reviews with optimistic updates */}
      <div style={{ marginBottom: '2rem' }}>
        <h3>Reviews</h3>
        {optimisticReviews.map((review) => (
          <div 
            key={review.id}
            style={{ 
              padding: '1rem',
              border: '1px solid #ddd',
              marginBottom: '0.5rem',
              opacity: review.pending ? 0.6 : 1,
            }}
          >
            <div>Rating: {review.rating} stars</div>
            <div>{review.comment}</div>
            {review.pending && (
              <div style={{ fontSize: '0.875rem', color: '#666' }}>
                Submitting...
              </div>
            )}
          </div>
        ))}
      </div>

      <form action={handleSubmit}>
        <input type="hidden" name="productId" value={productId} />

        {state?.error && (
          <div style={{ color: 'red', marginBottom: '1rem' }}>
            {state.error}
          </div>
        )}
        
        <div>
          <label htmlFor="rating">Rating:</label>
          <select id="rating" name="rating" defaultValue="5" required>
            {[1, 2, 3, 4, 5].map((n) => (
              <option key={n} value={n}>{n} stars</option>
            ))}
          </select>
        </div>
        
        <div>
          <label htmlFor="comment">Comment:</label>
          <textarea
            id="comment"
            name="comment"
            rows={4}
            required
            minLength={10}
          />
        </div>
        
        <SubmitButton />
      </form>
    </div>
  );
}
```

### Verification: Optimistic Updates Working

**Browser Behavior**:
- Fill out form with rating 5 and comment "Excellent product!"
- Click submit
- **Review appears immediately** with "Submitting..." label
- After server responds (~100ms), "Submitting..." disappears
- Review remains visible

**React DevTools - Components Tab**:
- `ReviewForm` component selected
- State shows optimistic review in array
- After server response, optimistic review replaced with real one

**Browser DevTools - Network Tab**:
```
POST /products/123
Status: 200 OK
Time: 98ms
```

**Expected vs. Actual Improvement**:
- **Before**: User waits for server response to see their review
- **After**: Review appears instantly, confirmed by server
- **Perceived performance**: Feels instant (0ms) vs. actual (100ms)
- **UX**: User can continue browsing immediately

### When to Use Server Actions vs. API Routes

| Scenario | Use Server Actions | Use API Routes |
|----------|-------------------|----------------|
| Form submissions | ✅ Yes | ❌ No |
| Mutations from UI | ✅ Yes | ❌ No |
| Progressive enhancement needed | ✅ Yes | ❌ No |
| External API consumption | ❌ No | ✅ Yes |
| Webhooks from third parties | ❌ No | ✅ Yes |
| Public API for mobile apps | ❌ No | ✅ Yes |
| Complex request/response headers | ❌ No | ✅ Yes |
| File uploads | ✅ Yes (with FormData) | ✅ Yes (both work) |

**Decision Framework**:

1. **Is this triggered by a user action in your UI?** → Server Action
2. **Does it need to work without JavaScript?** → Server Action
3. **Is it called by external systems?** → API Route
4. **Do you need custom HTTP headers/status codes?** → API Route
5. **Is it a simple mutation?** → Server Action
6. **Is it a complex multi-step process?** → API Route

### Limitation Preview

Server Actions are powerful, but they still require careful error handling and validation. In the next section, we'll explore how to handle errors gracefully and validate data comprehensively.

## Form handling with progressive enhancement

## The Problem: Forms That Break Gracefully

We've built a form with Server Actions, but there are still edge cases where things can go wrong:

1. **Network failures**: What if the request times out?
2. **Validation errors**: How do we preserve user input?
3. **Concurrent submissions**: What if the user clicks submit twice?
4. **Accessibility**: Is the form usable with keyboard and screen readers?

Let's build a production-ready form that handles all these cases.

### Iteration 4: Comprehensive Form Handling

**Current limitation**: Our form loses user input on validation errors, doesn't prevent double submissions, and lacks proper accessibility attributes.

**New scenario**: What happens when validation fails or the network is slow?

```typescript
// src/app/actions/reviews.ts
'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';
import { z } from 'zod';

const reviewSchema = z.object({
  productId: z.string().min(1, 'Product ID is required'),
  rating: z.coerce.number().min(1).max(5, 'Rating must be between 1 and 5'),
  comment: z.string()
    .min(10, 'Comment must be at least 10 characters')
    .max(500, 'Comment must not exceed 500 characters'),
});

export type ReviewFormState = {
  success: boolean;
  error?: string;
  fieldErrors?: {
    rating?: string[];
    comment?: string[];
  };
  // Preserve user input on error
  values?: {
    rating: number;
    comment: string;
  };
};

export async function submitReview(
  prevState: ReviewFormState | null,
  formData: FormData
): Promise<ReviewFormState> {
  // Simulate network delay for testing
  await new Promise(resolve => setTimeout(resolve, 1000));

  const rawData = {
    productId: formData.get('productId') as string,
    rating: formData.get('rating'),
    comment: formData.get('comment') as string,
  };

  const validation = reviewSchema.safeParse(rawData);

  if (!validation.success) {
    const fieldErrors = validation.error.flatten().fieldErrors;
    return {
      success: false,
      error: 'Please correct the errors below',
      fieldErrors,
      // Preserve user input
      values: {
        rating: Number(rawData.rating) || 5,
        comment: rawData.comment,
      },
    };
  }

  const { productId, rating, comment } = validation.data;

  try {
    const userId = 'user123';

    const review = await db.reviews.create({
      productId,
      userId,
      rating,
      comment,
    });

    revalidatePath(`/products/${productId}`);

    return {
      success: true,
    };
  } catch (error) {
    console.error('Review submission error:', error);
    return {
      success: false,
      error: 'Failed to submit review. Please try again.',
      values: {
        rating,
        comment,
      },
    };
  }
}
```

Now build a comprehensive form component:

```tsx
// src/app/products/[id]/ReviewForm.tsx
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { useEffect, useRef } from 'react';
import { submitReview, type ReviewFormState } from '@/app/actions/reviews';

function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button 
      type="submit" 
      disabled={pending}
      aria-disabled={pending}
    >
      {pending ? 'Submitting...' : 'Submit Review'}
    </button>
  );
}

export function ReviewForm({ productId }: { productId: string }) {
  const [state, formAction] = useFormState<ReviewFormState | null>(
    submitReview,
    null
  );
  const formRef = useRef<HTMLFormElement>(null);
  const commentRef = useRef<HTMLTextAreaElement>(null);

  // Reset form on successful submission
  useEffect(() => {
    if (state?.success) {
      formRef.current?.reset();
      // Focus on comment field for next review
      commentRef.current?.focus();
    }
  }, [state?.success]);

  // Focus on first error field
  useEffect(() => {
    if (state?.fieldErrors) {
      const firstErrorField = state.fieldErrors.rating 
        ? 'rating' 
        : 'comment';
      const element = formRef.current?.elements.namedItem(firstErrorField);
      if (element instanceof HTMLElement) {
        element.focus();
      }
    }
  }, [state?.fieldErrors]);

  return (
    <form 
      ref={formRef}
      action={formAction}
      aria-describedby={state?.error ? 'form-error' : undefined}
    >
      <input type="hidden" name="productId" value={productId} />

      {/* Global error message */}
      {state?.error && !state.success && (
        <div 
          id="form-error"
          role="alert"
          style={{ 
            color: 'red', 
            marginBottom: '1rem',
            padding: '0.75rem',
            border: '1px solid red',
            borderRadius: '4px',
            backgroundColor: '#fee',
          }}
        >
          {state.error}
        </div>
      )}

      {/* Success message */}
      {state?.success && (
        <div 
          role="status"
          style={{ 
            color: 'green', 
            marginBottom: '1rem',
            padding: '0.75rem',
            border: '1px solid green',
            borderRadius: '4px',
            backgroundColor: '#efe',
          }}
        >
          Review submitted successfully! Thank you for your feedback.
        </div>
      )}
      
      {/* Rating field */}
      <div style={{ marginBottom: '1rem' }}>
        <label htmlFor="rating">
          Rating: <span aria-label="required">*</span>
        </label>
        <select 
          id="rating" 
          name="rating" 
          defaultValue={state?.values?.rating || 5}
          required
          aria-required="true"
          aria-invalid={!!state?.fieldErrors?.rating}
          aria-describedby={
            state?.fieldErrors?.rating ? 'rating-error' : undefined
          }
        >
          {[1, 2, 3, 4, 5].map((n) => (
            <option key={n} value={n}>{n} stars</option>
          ))}
        </select>
        {state?.fieldErrors?.rating && (
          <p 
            id="rating-error"
            role="alert"
            style={{ 
              color: 'red', 
              fontSize: '0.875rem',
              marginTop: '0.25rem',
            }}
          >
            {state.fieldErrors.rating[0]}
          </p>
        )}
      </div>
      
      {/* Comment field */}
      <div style={{ marginBottom: '1rem' }}>
        <label htmlFor="comment">
          Comment: <span aria-label="required">*</span>
        </label>
        <textarea
          ref={commentRef}
          id="comment"
          name="comment"
          rows={4}
          required
          minLength={10}
          maxLength={500}
          defaultValue={state?.values?.comment || ''}
          aria-required="true"
          aria-invalid={!!state?.fieldErrors?.comment}
          aria-describedby={
            state?.fieldErrors?.comment 
              ? 'comment-error comment-hint' 
              : 'comment-hint'
          }
          style={{ width: '100%' }}
        />
        <p 
          id="comment-hint"
          style={{ 
            fontSize: '0.875rem', 
            color: '#666',
            marginTop: '0.25rem',
          }}
        >
          Minimum 10 characters, maximum 500 characters
        </p>
        {state?.fieldErrors?.comment && (
          <p 
            id="comment-error"
            role="alert"
            style={{ 
              color: 'red', 
              fontSize: '0.875rem',
              marginTop: '0.25rem',
            }}
          >
            {state.fieldErrors.comment[0]}
          </p>
        )}
      </div>
      
      <SubmitButton />
    </form>
  );
}
```

### Verification: Comprehensive Error Handling

**Test 1: Validation Error**

**Browser Behavior**:
- Enter rating: 5
- Enter comment: "Bad" (too short)
- Click submit
- Wait 1 second (simulated delay)
- Error message appears: "Please correct the errors below"
- Field-specific error: "Comment must be at least 10 characters"
- **User input preserved**: "Bad" still in textarea
- **Focus moved to comment field** automatically

**Browser Console Output**:
```
(No errors - handled gracefully)
```

**Accessibility Test (Screen Reader)**:
```
"Alert: Please correct the errors below"
"Comment, required, invalid, edit text"
"Alert: Comment must be at least 10 characters"
```

**Test 2: Network Failure Simulation**

**Browser DevTools - Network Tab**:
- Throttle to "Offline"
- Fill out form correctly
- Click submit

**Browser Behavior**:
- Button shows "Submitting..."
- After timeout (~30 seconds), error appears
- User input preserved
- Can retry submission

**Test 3: Double Submission Prevention**

**Browser Behavior**:
- Fill out form
- Click submit button rapidly 5 times
- Button becomes disabled after first click
- Only one request sent

**Browser DevTools - Network Tab**:
```
POST /products/123 (only one request)
Status: 200 OK
```

**Expected vs. Actual Improvement**:
- **Before**: Lost user input on error, no accessibility, double submissions possible
- **After**: Input preserved, fully accessible, double submission prevented
- **Accessibility**: WCAG 2.1 AA compliant
- **UX**: Clear error messages, automatic focus management

### Progressive Enhancement in Action

Let's verify the form works without JavaScript:

**Test: JavaScript Disabled**

**Browser DevTools**:
- Settings → Disable JavaScript
- Refresh page

**Browser Behavior**:
- Fill out form with invalid data (comment too short)
- Click submit
- **Page reloads** (full navigation)
- Error message appears on reloaded page
- **User input preserved** in form fields
- Can correct and resubmit

**Server Terminal Output**:
```
POST /products/123 (Server Action: submitReview)
Validation failed: comment too short
Returning HTML with form state
200 OK in 15ms
```

**How it works**:
1. Form submits as standard HTML POST
2. Server processes Server Action
3. Server returns new HTML with error state
4. Browser displays reloaded page with errors
5. Form fields populated with previous values

This is **progressive enhancement**: the form works without JavaScript, but provides a better experience with it.

### Iteration 5: Client-Side Validation for Instant Feedback

While server-side validation is essential for security, we can add client-side validation for better UX.

```tsx
// src/app/products/[id]/ReviewForm.tsx
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { useEffect, useRef, useState } from 'react';
import { submitReview, type ReviewFormState } from '@/app/actions/reviews';

function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button 
      type="submit" 
      disabled={pending}
      aria-disabled={pending}
    >
      {pending ? 'Submitting...' : 'Submit Review'}
    </button>
  );
}

export function ReviewForm({ productId }: { productId: string }) {
  const [state, formAction] = useFormState<ReviewFormState | null>(
    submitReview,
    null
  );
  const formRef = useRef<HTMLFormElement>(null);
  const commentRef = useRef<HTMLTextAreaElement>(null);
  
  // Client-side validation state
  const [commentError, setCommentError] = useState<string | null>(null);
  const [commentLength, setCommentLength] = useState(0);

  useEffect(() => {
    if (state?.success) {
      formRef.current?.reset();
      commentRef.current?.focus();
      setCommentLength(0);
      setCommentError(null);
    }
  }, [state?.success]);

  useEffect(() => {
    if (state?.fieldErrors) {
      const firstErrorField = state.fieldErrors.rating 
        ? 'rating' 
        : 'comment';
      const element = formRef.current?.elements.namedItem(firstErrorField);
      if (element instanceof HTMLElement) {
        element.focus();
      }
    }
  }, [state?.fieldErrors]);

  // Client-side validation on blur
  const handleCommentBlur = (e: React.FocusEvent<HTMLTextAreaElement>) => {
    const value = e.target.value;
    if (value.length > 0 && value.length < 10) {
      setCommentError('Comment must be at least 10 characters');
    } else if (value.length > 500) {
      setCommentError('Comment must not exceed 500 characters');
    } else {
      setCommentError(null);
    }
  };

  const handleCommentChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    setCommentLength(e.target.value.length);
    // Clear error as user types
    if (commentError && e.target.value.length >= 10) {
      setCommentError(null);
    }
  };

  // Use server error if present, otherwise client error
  const displayCommentError = state?.fieldErrors?.comment?.[0] || commentError;

  return (
    <form 
      ref={formRef}
      action={formAction}
      noValidate // Disable browser validation, use our own
      aria-describedby={state?.error ? 'form-error' : undefined}
    >
      <input type="hidden" name="productId" value={productId} />

      {state?.error && !state.success && (
        <div 
          id="form-error"
          role="alert"
          style={{ 
            color: 'red', 
            marginBottom: '1rem',
            padding: '0.75rem',
            border: '1px solid red',
            borderRadius: '4px',
            backgroundColor: '#fee',
          }}
        >
          {state.error}
        </div>
      )}

      {state?.success && (
        <div 
          role="status"
          style={{ 
            color: 'green', 
            marginBottom: '1rem',
            padding: '0.75rem',
            border: '1px solid green',
            borderRadius: '4px',
            backgroundColor: '#efe',
          }}
        >
          Review submitted successfully! Thank you for your feedback.
        </div>
      )}
      
      <div style={{ marginBottom: '1rem' }}>
        <label htmlFor="rating">
          Rating: <span aria-label="required">*</span>
        </label>
        <select 
          id="rating" 
          name="rating" 
          defaultValue={state?.values?.rating || 5}
          required
          aria-required="true"
          aria-invalid={!!state?.fieldErrors?.rating}
          aria-describedby={
            state?.fieldErrors?.rating ? 'rating-error' : undefined
          }
        >
          {[1, 2, 3, 4, 5].map((n) => (
            <option key={n} value={n}>{n} stars</option>
          ))}
        </select>
        {state?.fieldErrors?.rating && (
          <p 
            id="rating-error"
            role="alert"
            style={{ 
              color: 'red', 
              fontSize: '0.875rem',
              marginTop: '0.25rem',
            }}
          >
            {state.fieldErrors.rating[0]}
          </p>
        )}
      </div>
      
      <div style={{ marginBottom: '1rem' }}>
        <label htmlFor="comment">
          Comment: <span aria-label="required">*</span>
        </label>
        <textarea
          ref={commentRef}
          id="comment"
          name="comment"
          rows={4}
          required
          minLength={10}
          maxLength={500}
          defaultValue={state?.values?.comment || ''}
          onBlur={handleCommentBlur}
          onChange={handleCommentChange}
          aria-required="true"
          aria-invalid={!!displayCommentError}
          aria-describedby={
            displayCommentError 
              ? 'comment-error comment-hint' 
              : 'comment-hint'
          }
          style={{ width: '100%' }}
        />
        <div style={{ 
          display: 'flex', 
          justifyContent: 'space-between',
          marginTop: '0.25rem',
        }}>
          <p 
            id="comment-hint"
            style={{ 
              fontSize: '0.875rem', 
              color: '#666',
            }}
          >
            Minimum 10 characters
          </p>
          <p style={{ 
            fontSize: '0.875rem', 
            color: commentLength > 500 ? 'red' : '#666',
          }}>
            {commentLength}/500
          </p>
        </div>
        {displayCommentError && (
          <p 
            id="comment-error"
            role="alert"
            style={{ 
              color: 'red', 
              fontSize: '0.875rem',
              marginTop: '0.25rem',
            }}
          >
            {displayCommentError}
          </p>
        )}
      </div>
      
      <SubmitButton />
    </form>
  );
}
```

### Verification: Client-Side Validation

**Browser Behavior**:
- Start typing in comment field: "Bad"
- Tab out of field (blur event)
- **Instant feedback**: "Comment must be at least 10 characters"
- Continue typing: "Bad product"
- Error disappears as soon as 10 characters reached
- Character counter updates in real-time: "11/500"

**React DevTools - Components Tab**:
- `ReviewForm` component selected
- State: `{ commentError: null, commentLength: 11 }`

**Expected vs. Actual Improvement**:
- **Before**: No feedback until form submission
- **After**: Instant feedback on blur, real-time character count
- **UX**: User knows requirements before submitting
- **Validation**: Client-side for UX, server-side for security

### When to Apply: Form Validation Strategy

**Client-Side Validation**:
- **What it optimizes for**: Instant user feedback, reduced server load
- **What it sacrifices**: Can be bypassed, requires duplicate logic
- **When to use**: Always, as a UX enhancement
- **When to avoid**: Never rely on it alone for security

**Server-Side Validation**:
- **What it optimizes for**: Security, data integrity
- **What it sacrifices**: Slower feedback (network round-trip)
- **When to use**: Always, as the source of truth
- **When to avoid**: Never skip it

**Decision Framework**:
1. **Always validate on server** (security requirement)
2. **Add client validation for common errors** (UX enhancement)
3. **Keep validation logic in sync** (use shared schemas when possible)
4. **Provide clear error messages** (tell users how to fix)
5. **Preserve user input on error** (don't make them retype)

## Error handling and validation

## The Problem: Production-Grade Error Handling

Our form handles basic validation, but production applications need to handle:

1. **Network errors**: Timeouts, connection failures
2. **Server errors**: Database failures, external API issues
3. **Rate limiting**: Preventing abuse
4. **Concurrent requests**: Handling race conditions
5. **Logging and monitoring**: Tracking errors for debugging

Let's build a production-ready error handling system.

### Iteration 6: Comprehensive Error Handling

**Current limitation**: Generic error messages don't help users understand what went wrong or how to fix it.

**New scenario**: What happens when the database is down, or the user is rate-limited?

```typescript
// src/lib/errors.ts
// Centralized error handling utilities

export class AppError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number = 500,
    public userMessage?: string
  ) {
    super(message);
    this.name = 'AppError';
  }
}

export class ValidationError extends AppError {
  constructor(message: string, public fieldErrors?: Record<string, string[]>) {
    super(message, 'VALIDATION_ERROR', 400, 'Please correct the errors below');
  }
}

export class RateLimitError extends AppError {
  constructor() {
    super(
      'Rate limit exceeded',
      'RATE_LIMIT_EXCEEDED',
      429,
      'Too many requests. Please try again in a few minutes.'
    );
  }
}

export class DatabaseError extends AppError {
  constructor(originalError: Error) {
    super(
      originalError.message,
      'DATABASE_ERROR',
      500,
      'We encountered a technical issue. Please try again later.'
    );
  }
}

export function handleError(error: unknown): AppError {
  if (error instanceof AppError) {
    return error;
  }

  if (error instanceof Error) {
    // Log unexpected errors for monitoring
    console.error('Unexpected error:', error);
    return new AppError(
      error.message,
      'INTERNAL_ERROR',
      500,
      'An unexpected error occurred. Please try again.'
    );
  }

  console.error('Unknown error:', error);
  return new AppError(
    'Unknown error',
    'UNKNOWN_ERROR',
    500,
    'An unexpected error occurred. Please try again.'
  );
}
```

Add rate limiting:

```typescript
// src/lib/rate-limit.ts
// Simple in-memory rate limiter
// In production, use Redis or a dedicated service

type RateLimitEntry = {
  count: number;
  resetAt: number;
};

const rateLimits = new Map<string, RateLimitEntry>();

export function checkRateLimit(
  identifier: string,
  maxRequests: number = 5,
  windowMs: number = 60000 // 1 minute
): boolean {
  const now = Date.now();
  const entry = rateLimits.get(identifier);

  if (!entry || now > entry.resetAt) {
    // First request or window expired
    rateLimits.set(identifier, {
      count: 1,
      resetAt: now + windowMs,
    });
    return true;
  }

  if (entry.count >= maxRequests) {
    // Rate limit exceeded
    return false;
  }

  // Increment count
  entry.count++;
  return true;
}

export function getRateLimitInfo(identifier: string): {
  remaining: number;
  resetAt: number;
} | null {
  const entry = rateLimits.get(identifier);
  if (!entry) return null;

  return {
    remaining: Math.max(0, 5 - entry.count),
    resetAt: entry.resetAt,
  };
}
```

Update the Server Action with comprehensive error handling:

```typescript
// src/app/actions/reviews.ts
'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';
import { z } from 'zod';
import { 
  ValidationError, 
  RateLimitError, 
  DatabaseError,
  handleError 
} from '@/lib/errors';
import { checkRateLimit } from '@/lib/rate-limit';

const reviewSchema = z.object({
  productId: z.string().min(1, 'Product ID is required'),
  rating: z.coerce.number().min(1).max(5, 'Rating must be between 1 and 5'),
  comment: z.string()
    .min(10, 'Comment must be at least 10 characters')
    .max(500, 'Comment must not exceed 500 characters')
    .refine(
      (val) => !val.toLowerCase().includes('spam'),
      'Comment contains prohibited content'
    ),
});

export type ReviewFormState = {
  success: boolean;
  error?: string;
  errorCode?: string;
  fieldErrors?: {
    rating?: string[];
    comment?: string[];
  };
  values?: {
    rating: number;
    comment: string;
  };
  retryAfter?: number; // For rate limiting
};

export async function submitReview(
  prevState: ReviewFormState | null,
  formData: FormData
): Promise<ReviewFormState> {
  try {
    // 1. Rate limiting
    const userId = 'user123'; // In production, get from session
    const rateLimitKey = `review:${userId}`;
    
    if (!checkRateLimit(rateLimitKey, 5, 60000)) {
      throw new RateLimitError();
    }

    // 2. Parse form data
    const rawData = {
      productId: formData.get('productId') as string,
      rating: formData.get('rating'),
      comment: formData.get('comment') as string,
    };

    // 3. Validate
    const validation = reviewSchema.safeParse(rawData);

    if (!validation.success) {
      const fieldErrors = validation.error.flatten().fieldErrors;
      throw new ValidationError('Validation failed', fieldErrors);
    }

    const { productId, rating, comment } = validation.data;

    // 4. Database operation with error handling
    let review;
    try {
      review = await db.reviews.create({
        productId,
        userId,
        rating,
        comment,
      });
    } catch (error) {
      throw new DatabaseError(error as Error);
    }

    // 5. Revalidate cache
    try {
      revalidatePath(`/products/${productId}`);
    } catch (error) {
      // Log but don't fail - cache revalidation is not critical
      console.error('Cache revalidation failed:', error);
    }

    // 6. Success response
    return {
      success: true,
    };

  } catch (error) {
    // Centralized error handling
    const appError = handleError(error);

    const response: ReviewFormState = {
      success: false,
      error: appError.userMessage || appError.message,
      errorCode: appError.code,
    };

    // Add field errors for validation errors
    if (error instanceof ValidationError && error.fieldErrors) {
      response.fieldErrors = error.fieldErrors;
      response.values = {
        rating: Number(formData.get('rating')) || 5,
        comment: formData.get('comment') as string,
      };
    }

    // Add retry info for rate limit errors
    if (error instanceof RateLimitError) {
      response.retryAfter = 60; // seconds
    }

    // Log error for monitoring (in production, send to error tracking service)
    console.error('Review submission error:', {
      code: appError.code,
      message: appError.message,
      userId: 'user123',
      timestamp: new Date().toISOString(),
    });

    return response;
  }
}
```

Update the form to handle different error types:

```tsx
// src/app/products/[id]/ReviewForm.tsx
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { useEffect, useRef, useState } from 'react';
import { submitReview, type ReviewFormState } from '@/app/actions/reviews';

function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button 
      type="submit" 
      disabled={pending}
      aria-disabled={pending}
      style={{
        padding: '0.5rem 1rem',
        backgroundColor: pending ? '#ccc' : '#007bff',
        color: 'white',
        border: 'none',
        borderRadius: '4px',
        cursor: pending ? 'not-allowed' : 'pointer',
      }}
    >
      {pending ? 'Submitting...' : 'Submit Review'}
    </button>
  );
}

function ErrorMessage({ state }: { state: ReviewFormState | null }) {
  if (!state?.error || state.success) return null;

  // Different styling based on error type
  const isRateLimit = state.errorCode === 'RATE_LIMIT_EXCEEDED';
  const isValidation = state.errorCode === 'VALIDATION_ERROR';

  return (
    <div 
      id="form-error"
      role="alert"
      style={{ 
        color: isRateLimit ? '#856404' : 'red',
        marginBottom: '1rem',
        padding: '0.75rem',
        border: `1px solid ${isRateLimit ? '#ffc107' : 'red'}`,
        borderRadius: '4px',
        backgroundColor: isRateLimit ? '#fff3cd' : '#fee',
      }}
    >
      <strong>{isRateLimit ? 'Rate Limit Exceeded' : 'Error'}:</strong>{' '}
      {state.error}
      {state.retryAfter && (
        <p style={{ marginTop: '0.5rem', fontSize: '0.875rem' }}>
          Please wait {state.retryAfter} seconds before trying again.
        </p>
      )}
    </div>
  );
}

export function ReviewForm({ productId }: { productId: string }) {
  const [state, formAction] = useFormState<ReviewFormState | null>(
    submitReview,
    null
  );
  const formRef = useRef<HTMLFormElement>(null);
  const commentRef = useRef<HTMLTextAreaElement>(null);
  const [commentLength, setCommentLength] = useState(0);

  useEffect(() => {
    if (state?.success) {
      formRef.current?.reset();
      commentRef.current?.focus();
      setCommentLength(0);
    }
  }, [state?.success]);

  useEffect(() => {
    if (state?.fieldErrors) {
      const firstErrorField = state.fieldErrors.rating 
        ? 'rating' 
        : 'comment';
      const element = formRef.current?.elements.namedItem(firstErrorField);
      if (element instanceof HTMLElement) {
        element.focus();
      }
    }
  }, [state?.fieldErrors]);

  const handleCommentChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    setCommentLength(e.target.value.length);
  };

  return (
    <form 
      ref={formRef}
      action={formAction}
      noValidate
      aria-describedby={state?.error ? 'form-error' : undefined}
    >
      <input type="hidden" name="productId" value={productId} />

      <ErrorMessage state={state} />

      {state?.success && (
        <div 
          role="status"
          style={{ 
            color: 'green', 
            marginBottom: '1rem',
            padding: '0.75rem',
            border: '1px solid green',
            borderRadius: '4px',
            backgroundColor: '#efe',
          }}
        >
          ✓ Review submitted successfully! Thank you for your feedback.
        </div>
      )}
      
      <div style={{ marginBottom: '1rem' }}>
        <label htmlFor="rating">
          Rating: <span aria-label="required">*</span>
        </label>
        <select 
          id="rating" 
          name="rating" 
          defaultValue={state?.values?.rating || 5}
          required
          aria-required="true"
          aria-invalid={!!state?.fieldErrors?.rating}
          aria-describedby={
            state?.fieldErrors?.rating ? 'rating-error' : undefined
          }
          style={{ 
            marginLeft: '0.5rem',
            padding: '0.25rem',
          }}
        >
          {[1, 2, 3, 4, 5].map((n) => (
            <option key={n} value={n}>{n} stars</option>
          ))}
        </select>
        {state?.fieldErrors?.rating && (
          <p 
            id="rating-error"
            role="alert"
            style={{ 
              color: 'red', 
              fontSize: '0.875rem',
              marginTop: '0.25rem',
            }}
          >
            {state.fieldErrors.rating[0]}
          </p>
        )}
      </div>
      
      <div style={{ marginBottom: '1rem' }}>
        <label htmlFor="comment" style={{ display: 'block', marginBottom: '0.25rem' }}>
          Comment: <span aria-label="required">*</span>
        </label>
        <textarea
          ref={commentRef}
          id="comment"
          name="comment"
          rows={4}
          required
          minLength={10}
          maxLength={500}
          defaultValue={state?.values?.comment || ''}
          onChange={handleCommentChange}
          aria-required="true"
          aria-invalid={!!state?.fieldErrors?.comment}
          aria-describedby={
            state?.fieldErrors?.comment 
              ? 'comment-error comment-hint' 
              : 'comment-hint'
          }
          style={{ 
            width: '100%',
            padding: '0.5rem',
            border: state?.fieldErrors?.comment ? '2px solid red' : '1px solid #ccc',
            borderRadius: '4px',
          }}
        />
        <div style={{ 
          display: 'flex', 
          justifyContent: 'space-between',
          marginTop: '0.25rem',
        }}>
          <p 
            id="comment-hint"
            style={{ 
              fontSize: '0.875rem', 
              color: '#666',
            }}
          >
            Minimum 10 characters
          </p>
          <p style={{ 
            fontSize: '0.875rem', 
            color: commentLength > 500 ? 'red' : '#666',
          }}>
            {commentLength}/500
          </p>
        </div>
        {state?.fieldErrors?.comment && (
          <p 
            id="comment-error"
            role="alert"
            style={{ 
              color: 'red', 
              fontSize: '0.875rem',
              marginTop: '0.25rem',
            }}
          >
            {state.fieldErrors.comment[0]}
          </p>
        )}
      </div>
      
      <SubmitButton />
    </form>
  );
}
```

### Verification: Production-Grade Error Handling

**Test 1: Rate Limiting**

**Browser Behavior**:
- Submit 5 reviews rapidly
- On 6th submission, see error:
  "Too many requests. Please wait 60 seconds before trying again."
- Error styled differently (yellow warning vs. red error)

**Server Terminal Output**:
```
POST /products/123 (submitReview) 201 OK in 12ms
POST /products/123 (submitReview) 201 OK in 11ms
POST /products/123 (submitReview) 201 OK in 13ms
POST /products/123 (submitReview) 201 OK in 12ms
POST /products/123 (submitReview) 201 OK in 14ms
POST /products/123 (submitReview) 429 Rate Limit Exceeded
Review submission error: {
  code: 'RATE_LIMIT_EXCEEDED',
  message: 'Rate limit exceeded',
  userId: 'user123',
  timestamp: '2025-01-15T10:35:00.000Z'
}
```

**Test 2: Validation Error**

**Browser Behavior**:
- Enter comment: "spam spam spam"
- Submit form
- Error: "Comment contains prohibited content"
- Field highlighted in red
- User input preserved

**Test 3: Database Error Simulation**

Temporarily break the database:

```typescript
// src/lib/db.ts - Simulate database failure
export const db = {
  reviews: {
    create: async () => {
      throw new Error('Database connection failed');
    },
  },
};
```

**Browser Behavior**:
- Submit valid review
- Error: "We encountered a technical issue. Please try again later."
- Generic message (doesn't expose internal details)

**Server Terminal Output**:
```
POST /products/123 (submitReview)
Unexpected error: Error: Database connection failed
Review submission error: {
  code: 'DATABASE_ERROR',
  message: 'Database connection failed',
  userId: 'user123',
  timestamp: '2025-01-15T10:36:00.000Z'
}
500 Internal Server Error
```

**Expected vs. Actual Improvement**:
- **Before**: Generic "error occurred" message for all failures
- **After**: Specific, actionable error messages
- **Security**: Internal errors don't leak implementation details
- **Monitoring**: All errors logged with context for debugging
- **UX**: Users know what went wrong and how to fix it

### Common Failure Modes and Their Signatures

#### Symptom: "Too many requests" error

**Browser behavior**: Yellow warning box with retry timer

**Console pattern**:
```
POST /api/reviews 429 Too Many Requests
```

**Server logs**:
```
Review submission error: { code: 'RATE_LIMIT_EXCEEDED', ... }
```

**Root cause**: User exceeded rate limit (5 requests per minute)

**Solution**: Wait for rate limit window to expire, or increase limit for authenticated users

#### Symptom: "Validation failed" with field-specific errors

**Browser behavior**: Red error box, specific fields highlighted

**Console pattern**:
```
POST /api/reviews 400 Bad Request
```

**Server logs**:
```
Review submission error: { code: 'VALIDATION_ERROR', fieldErrors: {...} }
```

**Root cause**: User input doesn't meet validation requirements

**Solution**: Fix the specific field errors shown

#### Symptom: "Technical issue" generic error

**Browser behavior**: Red error box with generic message

**Console pattern**:
```
POST /api/reviews 500 Internal Server Error
```

**Server logs**:
```
Unexpected error: Error: Database connection failed
Review submission error: { code: 'DATABASE_ERROR', ... }
```

**Root cause**: Server-side failure (database, external API, etc.)

**Solution**: Check server logs, verify database connection, retry request

### When to Apply: Error Handling Strategy

**Client-Side Error Handling**:
- **What it optimizes for**: Instant feedback, reduced server load
- **What it sacrifices**: Can't catch server-side errors
- **When to use**: Input validation, format checking
- **When to avoid**: Security-critical validation, business logic

**Server-Side Error Handling**:
- **What it optimizes for**: Security, data integrity, comprehensive error tracking
- **What it sacrifices**: Slower feedback (network round-trip)
- **When to use**: Always, as the authoritative error handler
- **When to avoid**: Never skip it

**Error Logging**:
- **What it optimizes for**: Debugging, monitoring, alerting
- **What it sacrifices**: Performance overhead, storage costs
- **When to use**: Production environments, unexpected errors
- **When to avoid**: Sensitive data (passwords, tokens)

**Decision Framework**:

1. **Validate on client for UX** (instant feedback)
2. **Validate on server for security** (authoritative)
3. **Use specific error types** (ValidationError, RateLimitError, etc.)
4. **Log errors with context** (user ID, timestamp, error code)
5. **Show user-friendly messages** (hide implementation details)
6. **Provide actionable guidance** (tell users how to fix)
7. **Monitor error rates** (alert on spikes)

## The Complete Journey - Chapter 18 Synthesis

## The Journey: From Insecure Client Code to Production-Ready Server Actions

Let's trace the evolution of our review submission system through each iteration:

| Iteration | Failure Mode | Technique Applied | Result | Key Improvement |
|-----------|--------------|-------------------|--------|-----------------|
| 0 | API key exposed in client code | None | Security breach | Baseline (insecure) |
| 1 | Secrets in client bundle | API Routes | Secrets protected | Server-side security |
| 2 | External API dependency | Database integration | Faster, more control | 10-20x performance |
| 3 | Form broken without JS | Server Actions | Progressive enhancement | Works without JS |
| 4 | Lost input on error | Form state preservation | Better UX | Input preserved |
| 5 | No instant feedback | Client-side validation | Faster feedback | Real-time validation |
| 6 | Generic error messages | Comprehensive error handling | Clear guidance | Production-ready |

### Final Implementation: Production-Ready Review System

Here's the complete, production-ready implementation with all improvements integrated:

**Project Structure**:
```
src/
├── app/
│   ├── products/
│   │   └── [id]/
│   │       ├── page.tsx
│   │       └── ReviewForm.tsx       ← Final form component
│   ├── actions/
│   │   └── reviews.ts               ← Server Actions
│   └── api/
│       └── reviews/
│           └── [id]/
│               └── route.ts         ← API routes for external access
├── lib/
│   ├── db.ts                        ← Database utilities
│   ├── errors.ts                    ← Error handling
│   └── rate-limit.ts                ← Rate limiting
└── types/
    └── review.ts                    ← Shared types
```

```typescript
// src/types/review.ts
export type Review = {
  id: string;
  productId: string;
  userId: string;
  rating: number;
  comment: string;
  createdAt: Date;
};

export type ReviewFormState = {
  success: boolean;
  error?: string;
  errorCode?: string;
  fieldErrors?: {
    rating?: string[];
    comment?: string[];
  };
  values?: {
    rating: number;
    comment: string;
  };
  retryAfter?: number;
};
```
