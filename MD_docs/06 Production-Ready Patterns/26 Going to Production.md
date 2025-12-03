# Chapter 26: Going to Production

## Environment configuration

## The Production Environment Problem

You've built a feature-complete application. It works perfectly on your laptop. You run `npm run build`, deploy to production, and... nothing works. The API calls fail. The authentication breaks. The analytics don't fire. The feature flags are stuck in development mode.

**This is the environment configuration problem**: Your application needs different settings in different environments, but you've hardcoded everything for local development.

Let's see this failure in action, then build a robust environment configuration system.

### Phase 1: The Reference Implementation - Hardcoded Configuration

We'll build a production-ready e-commerce checkout flow that needs different configurations across environments:

- **Development**: Local API, test payment keys, verbose logging
- **Staging**: Staging API, test payment keys, moderate logging
- **Production**: Production API, live payment keys, minimal logging

Here's the naive approach that will fail in production:

```tsx
// src/app/checkout/page.tsx
'use client';

import { useState } from 'react';

export default function CheckoutPage() {
  const [isProcessing, setIsProcessing] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleCheckout() {
    setIsProcessing(true);
    setError(null);

    try {
      // Hardcoded configuration - works in development
      const response = await fetch('http://localhost:3001/api/checkout', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-API-Key': 'dev_test_key_12345',
        },
        body: JSON.stringify({
          items: [{ id: 'prod_1', quantity: 2 }],
          paymentMethod: 'test_card_4242',
        }),
      });

      if (!response.ok) {
        throw new Error('Checkout failed');
      }

      const data = await response.json();
      console.log('Checkout successful:', data);
      
      // Hardcoded analytics
      window.gtag?.('event', 'purchase', {
        transaction_id: data.orderId,
        value: data.total,
      });

    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
      console.error('Checkout error:', err);
    } finally {
      setIsProcessing(false);
    }
  }

  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Checkout</h1>
      
      {error && (
        <div className="bg-red-50 border border-red-200 text-red-800 px-4 py-3 rounded mb-4">
          {error}
        </div>
      )}

      <button
        onClick={handleCheckout}
        disabled={isProcessing}
        className="bg-blue-600 text-white px-6 py-3 rounded-lg disabled:opacity-50"
      >
        {isProcessing ? 'Processing...' : 'Complete Purchase'}
      </button>
    </div>
  );
}
```

### Iteration 0: The Production Deployment Failure

You deploy this to production. Let's see what happens:

**Browser Console**:
```
POST https://your-app.com/checkout net::ERR_FAILED
Checkout error: TypeError: Failed to fetch
```

**Network Tab**:
- Request to `http://localhost:3001/api/checkout`
- Status: (failed) net::ERR_CONNECTION_REFUSED
- No response received

**User Experience**:
- Button shows "Processing..." briefly
- Error message appears: "Checkout failed"
- No purchase is completed
- No analytics event fires

### Diagnostic Analysis: Reading the Production Failure

**What the user experiences**:
- Expected: Successful checkout with confirmation
- Actual: Immediate error, no purchase processed

**What the console reveals**:
- Key indicator: `net::ERR_FAILED` on fetch to localhost
- Error location: The hardcoded `http://localhost:3001` URL
- Root cause: Trying to connect to localhost from production server

**What the Network tab shows**:
- Request pattern: Single failed request to localhost
- No retry attempts
- Browser can't resolve localhost in production context

**Root cause identified**: Hardcoded development URLs and API keys don't work in production.

**Why the current approach can't solve this**: You can't change the code for each environment. You need configuration that changes automatically based on where the app is running.

**What we need**: Environment-specific configuration that's:
1. Secure (no secrets in client code)
2. Type-safe (catch configuration errors at build time)
3. Environment-aware (automatically uses correct values)
4. Validated (fails fast if misconfigured)

## Building a Robust Environment Configuration System

### The Environment Variable Foundation

Next.js provides built-in environment variable support with important security distinctions:

**Server-side variables** (private):
- Available only in Server Components and API Routes
- Never sent to the browser
- Perfect for API keys, database URLs, secrets

**Client-side variables** (public):
- Must be prefixed with `NEXT_PUBLIC_`
- Bundled into the client JavaScript
- Visible to anyone who inspects your code
- Use only for non-sensitive configuration

Let's build our configuration system:

```bash
# .env.local (for local development - never commit this)
# This file is gitignored by default

# Server-side only (secure)
STRIPE_SECRET_KEY=sk_test_51abc123...
DATABASE_URL=postgresql://localhost:5432/myapp
API_SECRET_KEY=dev_secret_key_12345

# Client-side (public - will be in browser bundle)
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_51abc123...
NEXT_PUBLIC_ANALYTICS_ID=G-XXXXXXXXXX
NEXT_PUBLIC_ENVIRONMENT=development
```

```bash
# .env.production (committed to repo - production defaults)
# These are overridden by actual secrets in deployment platform

# Server-side placeholders (real values set in Vercel/deployment platform)
STRIPE_SECRET_KEY=sk_live_placeholder
DATABASE_URL=postgresql://placeholder
API_SECRET_KEY=placeholder

# Client-side production values (safe to commit)
NEXT_PUBLIC_API_URL=https://api.yourapp.com
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_placeholder
NEXT_PUBLIC_ANALYTICS_ID=G-PRODUCTION123
NEXT_PUBLIC_ENVIRONMENT=production
```

### Type-Safe Environment Configuration

Raw `process.env` access is error-prone. Let's create a type-safe configuration layer:

```typescript
// src/config/env.ts
// Type-safe environment variable access with validation

import { z } from 'zod';

/**
 * Schema for server-side environment variables
 * These are NEVER exposed to the client
 */
const serverEnvSchema = z.object({
  STRIPE_SECRET_KEY: z.string().min(1, 'Stripe secret key is required'),
  DATABASE_URL: z.string().url('Database URL must be valid'),
  API_SECRET_KEY: z.string().min(1, 'API secret key is required'),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
});

/**
 * Schema for client-side environment variables
 * These ARE exposed to the client (must be prefixed with NEXT_PUBLIC_)
 */
const clientEnvSchema = z.object({
  NEXT_PUBLIC_API_URL: z.string().url('API URL must be valid'),
  NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: z.string().min(1, 'Stripe publishable key is required'),
  NEXT_PUBLIC_ANALYTICS_ID: z.string().optional(),
  NEXT_PUBLIC_ENVIRONMENT: z.enum(['development', 'staging', 'production']).default('development'),
});

/**
 * Validate and parse server environment variables
 * Call this in Server Components or API Routes only
 */
export function getServerEnv() {
  const parsed = serverEnvSchema.safeParse(process.env);
  
  if (!parsed.success) {
    console.error('❌ Invalid server environment variables:', parsed.error.flatten().fieldErrors);
    throw new Error('Invalid server environment configuration');
  }
  
  return parsed.data;
}

/**
 * Validate and parse client environment variables
 * Safe to call anywhere (client or server)
 */
export function getClientEnv() {
  const parsed = clientEnvSchema.safeParse({
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
    NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY,
    NEXT_PUBLIC_ANALYTICS_ID: process.env.NEXT_PUBLIC_ANALYTICS_ID,
    NEXT_PUBLIC_ENVIRONMENT: process.env.NEXT_PUBLIC_ENVIRONMENT,
  });
  
  if (!parsed.success) {
    console.error('❌ Invalid client environment variables:', parsed.error.flatten().fieldErrors);
    throw new Error('Invalid client environment configuration');
  }
  
  return parsed.data;
}

/**
 * Type-safe client environment access
 * Use this in Client Components
 */
export const clientEnv = getClientEnv();

/**
 * Helper to check current environment
 */
export const isDevelopment = clientEnv.NEXT_PUBLIC_ENVIRONMENT === 'development';
export const isStaging = clientEnv.NEXT_PUBLIC_ENVIRONMENT === 'staging';
export const isProduction = clientEnv.NEXT_PUBLIC_ENVIRONMENT === 'production';
```

### Iteration 1: Environment-Aware Checkout

Now let's refactor our checkout to use environment configuration:

```tsx
// src/app/checkout/page.tsx
'use client';

import { useState } from 'react';
import { clientEnv, isProduction } from '@/config/env';

export default function CheckoutPage() {
  const [isProcessing, setIsProcessing] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function handleCheckout() {
    setIsProcessing(true);
    setError(null);

    try {
      // Environment-aware API URL
      const response = await fetch(`${clientEnv.NEXT_PUBLIC_API_URL}/api/checkout`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          items: [{ id: 'prod_1', quantity: 2 }],
          // Use environment-specific payment key
          stripeKey: clientEnv.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY,
        }),
      });

      if (!response.ok) {
        throw new Error('Checkout failed');
      }

      const data = await response.json();
      
      // Conditional logging based on environment
      if (!isProduction) {
        console.log('Checkout successful:', data);
      }
      
      // Environment-aware analytics
      if (clientEnv.NEXT_PUBLIC_ANALYTICS_ID && window.gtag) {
        window.gtag('event', 'purchase', {
          transaction_id: data.orderId,
          value: data.total,
        });
      }

    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
      
      // Detailed error logging in non-production
      if (!isProduction) {
        console.error('Checkout error:', err);
      }
    } finally {
      setIsProcessing(false);
    }
  }

  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Checkout</h1>
      
      {/* Development-only environment indicator */}
      {!isProduction && (
        <div className="bg-yellow-50 border border-yellow-200 text-yellow-800 px-4 py-2 rounded mb-4 text-sm">
          Environment: {clientEnv.NEXT_PUBLIC_ENVIRONMENT} | API: {clientEnv.NEXT_PUBLIC_API_URL}
        </div>
      )}
      
      {error && (
        <div className="bg-red-50 border border-red-200 text-red-800 px-4 py-3 rounded mb-4">
          {error}
        </div>
      )}

      <button
        onClick={handleCheckout}
        disabled={isProcessing}
        className="bg-blue-600 text-white px-6 py-3 rounded-lg disabled:opacity-50"
      >
        {isProcessing ? 'Processing...' : 'Complete Purchase'}
      </button>
    </div>
  );
}
```

**Verification**: Deploy to production with proper environment variables set:

**Browser Console** (production):
```
(No checkout logs - production mode)
```

**Network Tab**:
- Request to `https://api.yourapp.com/api/checkout`
- Status: 200 OK
- Response received successfully

**User Experience**:
- Button processes correctly
- Success state shown
- Analytics event fires
- No environment indicator visible

**Expected vs. Actual improvement**:
- Before: 100% failure rate in production (localhost connection refused)
- After: Successful API calls using correct production URL
- Before: Test API keys exposed in production
- After: Environment-specific keys used correctly
- Before: Verbose logging in production
- After: Minimal logging, detailed only in development

### Server-Side Environment Configuration

For API routes and Server Components, we need secure access to server-side variables:

```typescript
// src/app/api/checkout/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerEnv } from '@/config/env';
import Stripe from 'stripe';

export async function POST(request: NextRequest) {
  const env = getServerEnv();
  
  // Initialize Stripe with server-side secret key
  const stripe = new Stripe(env.STRIPE_SECRET_KEY, {
    apiVersion: '2023-10-16',
  });

  try {
    const body = await request.json();
    
    // Create payment intent with secret key
    const paymentIntent = await stripe.paymentIntents.create({
      amount: 2000,
      currency: 'usd',
      metadata: {
        environment: env.NODE_ENV,
      },
    });

    // Log only in development
    if (env.NODE_ENV === 'development') {
      console.log('Payment intent created:', paymentIntent.id);
    }

    return NextResponse.json({
      orderId: paymentIntent.id,
      total: paymentIntent.amount,
      clientSecret: paymentIntent.client_secret,
    });

  } catch (error) {
    console.error('Checkout error:', error);
    
    return NextResponse.json(
      { error: 'Checkout failed' },
      { status: 500 }
    );
  }
}
```

### Build-Time Environment Validation

Catch configuration errors before deployment:

```typescript
// src/config/validate-env.ts
// Run this during build to catch configuration errors early

import { getClientEnv, getServerEnv } from './env';

/**
 * Validate environment configuration at build time
 * Add this to your build script: "build": "node -r ./src/config/validate-env.ts next build"
 */
export function validateEnvironment() {
  console.log('🔍 Validating environment configuration...');
  
  try {
    // Validate client environment (always available)
    const clientEnv = getClientEnv();
    console.log('✅ Client environment valid');
    console.log(`   Environment: ${clientEnv.NEXT_PUBLIC_ENVIRONMENT}`);
    console.log(`   API URL: ${clientEnv.NEXT_PUBLIC_API_URL}`);
    
    // Validate server environment (only in Node.js context)
    if (typeof window === 'undefined') {
      const serverEnv = getServerEnv();
      console.log('✅ Server environment valid');
      console.log(`   Node environment: ${serverEnv.NODE_ENV}`);
    }
    
    console.log('✅ Environment configuration validated successfully\n');
    
  } catch (error) {
    console.error('❌ Environment validation failed:');
    console.error(error);
    process.exit(1);
  }
}

// Run validation if this file is executed directly
if (require.main === module) {
  validateEnvironment();
}
```

```json
// package.json
{
  "scripts": {
    "dev": "next dev",
    "build": "node -r ./src/config/validate-env.ts && next build",
    "start": "next start",
    "validate-env": "node -r ./src/config/validate-env.ts"
  }
}
```

### Common Failure Mode: Missing Environment Variables

**Symptom**: Build succeeds but app crashes at runtime

**Browser Console**:
```
Error: Invalid client environment configuration
  at getClientEnv (env.ts:45)
```

**Terminal Output** (during build):
```
❌ Invalid client environment variables: {
  NEXT_PUBLIC_API_URL: ['Required']
}
```

**Root cause**: Environment variable not set in deployment platform

**Solution**: 
1. Check deployment platform environment variables
2. Ensure all required variables are set
3. Verify variable names match exactly (including `NEXT_PUBLIC_` prefix)
4. Redeploy after adding missing variables

### When to Apply This Solution

**What it optimizes for**:
- Security (secrets never in client code)
- Type safety (catch errors at build time)
- Environment flexibility (same code, different configs)
- Developer experience (autocomplete, validation)

**What it sacrifices**:
- Initial setup complexity
- Additional validation code
- Build-time overhead (minimal)

**When to choose this approach**:
- Any production application
- Multiple deployment environments
- Team collaboration (prevents configuration drift)
- Applications with secrets or API keys

**When to avoid this approach**:
- Simple static sites with no backend
- Prototypes with no deployment plans
- Single-environment applications (rare)

**Code characteristics**:
- Setup: ~100 lines of configuration code
- Maintenance: Low (add variables as needed)
- Performance: Zero runtime impact (build-time validation)

## Feature flags with simple patterns

## The Feature Flag Problem

You've built a new checkout flow. It's ready for testing, but you don't want to deploy it to all users yet. You want to:

1. Test it with internal users first
2. Gradually roll it out to 10%, then 50%, then 100% of users
3. Instantly disable it if something goes wrong
4. A/B test it against the old flow

**Without feature flags**, you'd need to:
- Maintain separate branches for each feature
- Deploy different code to different environments
- Manually revert deployments when issues arise
- Can't test in production without affecting all users

Let's build a simple, effective feature flag system.

### Phase 1: The Hardcoded Feature Toggle

Here's the naive approach - a boolean constant:

```tsx
// src/app/checkout/page.tsx
'use client';

import { useState } from 'react';

// Hardcoded feature flag
const USE_NEW_CHECKOUT = false;

export default function CheckoutPage() {
  if (USE_NEW_CHECKOUT) {
    return <NewCheckoutFlow />;
  }
  
  return <OldCheckoutFlow />;
}

function NewCheckoutFlow() {
  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">New Checkout (Beta)</h1>
      <p>Improved checkout experience with one-click payment</p>
    </div>
  );
}

function OldCheckoutFlow() {
  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Checkout</h1>
      <p>Standard checkout flow</p>
    </div>
  );
}
```

### The Deployment Problem

You want to enable the new checkout for internal testing. Your options:

1. **Change the constant and deploy**: Now ALL users see it (too risky)
2. **Use environment variables**: Can't change without redeploying
3. **Maintain separate branches**: Merge conflicts, deployment complexity

**What we need**: Feature flags that can be:
- Changed without redeploying
- Targeted to specific users or percentages
- Instantly toggled on/off
- Tracked and audited

## Building a Simple Feature Flag System

We'll build a pragmatic system that doesn't require external services for basic use cases.

### The Feature Flag Configuration

```typescript
// src/config/features.ts
// Simple feature flag system with multiple strategies

export type FeatureFlagStrategy = 
  | { type: 'boolean'; enabled: boolean }
  | { type: 'percentage'; rollout: number } // 0-100
  | { type: 'userList'; allowedUsers: string[] }
  | { type: 'environment'; environments: string[] };

export interface FeatureFlag {
  key: string;
  name: string;
  description: string;
  strategy: FeatureFlagStrategy;
  createdAt: string;
  updatedAt: string;
}

/**
 * Feature flag configuration
 * In production, this would come from a database or API
 * For now, we'll use a simple in-memory configuration
 */
export const featureFlags: Record<string, FeatureFlag> = {
  'new-checkout': {
    key: 'new-checkout',
    name: 'New Checkout Flow',
    description: 'Improved one-click checkout experience',
    strategy: { type: 'percentage', rollout: 10 }, // 10% of users
    createdAt: '2024-01-15T10:00:00Z',
    updatedAt: '2024-01-15T10:00:00Z',
  },
  
  'express-shipping': {
    key: 'express-shipping',
    name: 'Express Shipping Option',
    description: 'Same-day delivery for eligible items',
    strategy: { type: 'environment', environments: ['development', 'staging'] },
    createdAt: '2024-01-10T10:00:00Z',
    updatedAt: '2024-01-10T10:00:00Z',
  },
  
  'admin-dashboard': {
    key: 'admin-dashboard',
    name: 'New Admin Dashboard',
    description: 'Redesigned admin interface',
    strategy: { 
      type: 'userList', 
      allowedUsers: ['admin@example.com', 'dev@example.com'] 
    },
    createdAt: '2024-01-01T10:00:00Z',
    updatedAt: '2024-01-01T10:00:00Z',
  },
};

/**
 * Get a feature flag by key
 */
export function getFeatureFlag(key: string): FeatureFlag | undefined {
  return featureFlags[key];
}

/**
 * Update a feature flag (in production, this would update the database)
 */
export function updateFeatureFlag(key: string, updates: Partial<FeatureFlag>): void {
  const flag = featureFlags[key];
  if (flag) {
    featureFlags[key] = {
      ...flag,
      ...updates,
      updatedAt: new Date().toISOString(),
    };
  }
}
```

### The Feature Flag Evaluation Engine

```typescript
// src/lib/feature-flags.ts
// Feature flag evaluation logic

import { getFeatureFlag, type FeatureFlagStrategy } from '@/config/features';
import { clientEnv } from '@/config/env';

/**
 * Context for evaluating feature flags
 */
export interface FeatureFlagContext {
  userId?: string;
  userEmail?: string;
  environment?: string;
}

/**
 * Generate a consistent hash for percentage-based rollouts
 * Same user always gets same result for same feature
 */
function hashString(str: string): number {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    const char = str.charCodeAt(i);
    hash = ((hash << 5) - hash) + char;
    hash = hash & hash; // Convert to 32-bit integer
  }
  return Math.abs(hash);
}

/**
 * Evaluate a feature flag strategy
 */
function evaluateStrategy(
  strategy: FeatureFlagStrategy,
  context: FeatureFlagContext
): boolean {
  switch (strategy.type) {
    case 'boolean':
      return strategy.enabled;
      
    case 'percentage': {
      // Need a user identifier for consistent rollout
      const identifier = context.userId || context.userEmail;
      if (!identifier) {
        return false; // Can't do percentage rollout without user ID
      }
      
      // Generate consistent hash and convert to percentage
      const hash = hashString(identifier);
      const userPercentage = hash % 100;
      
      return userPercentage < strategy.rollout;
    }
    
    case 'userList': {
      const userEmail = context.userEmail;
      if (!userEmail) {
        return false;
      }
      
      return strategy.allowedUsers.includes(userEmail);
    }
    
    case 'environment': {
      const environment = context.environment || clientEnv.NEXT_PUBLIC_ENVIRONMENT;
      return strategy.environments.includes(environment);
    }
    
    default:
      return false;
  }
}

/**
 * Check if a feature flag is enabled for the given context
 */
export function isFeatureEnabled(
  flagKey: string,
  context: FeatureFlagContext = {}
): boolean {
  const flag = getFeatureFlag(flagKey);
  
  if (!flag) {
    // Flag doesn't exist - default to disabled
    console.warn(`Feature flag "${flagKey}" not found`);
    return false;
  }
  
  return evaluateStrategy(flag.strategy, context);
}

/**
 * React hook for feature flags
 */
export function useFeatureFlag(
  flagKey: string,
  context: FeatureFlagContext = {}
): boolean {
  // In a real app, this would subscribe to flag updates
  // For now, we evaluate once
  return isFeatureEnabled(flagKey, context);
}
```

### Iteration 1: Feature-Flagged Checkout

Now let's use feature flags in our checkout:

```tsx
// src/app/checkout/page.tsx
'use client';

import { useFeatureFlag } from '@/lib/feature-flags';
import { useUser } from '@/hooks/useUser'; // Assume this exists

export default function CheckoutPage() {
  const { user } = useUser();
  
  // Evaluate feature flag with user context
  const useNewCheckout = useFeatureFlag('new-checkout', {
    userId: user?.id,
    userEmail: user?.email,
  });
  
  if (useNewCheckout) {
    return <NewCheckoutFlow />;
  }
  
  return <OldCheckoutFlow />;
}

function NewCheckoutFlow() {
  return (
    <div className="max-w-2xl mx-auto p-6">
      <div className="bg-blue-50 border border-blue-200 text-blue-800 px-4 py-2 rounded mb-4 text-sm">
        ✨ You're using our new checkout experience!
      </div>
      <h1 className="text-3xl font-bold mb-6">Express Checkout</h1>
      <p>One-click payment with saved cards</p>
      {/* New checkout implementation */}
    </div>
  );
}

function OldCheckoutFlow() {
  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Checkout</h1>
      <p>Standard checkout flow</p>
      {/* Original checkout implementation */}
    </div>
  );
}
```

**Verification**: With 10% rollout configured:

**User A** (userId: "user_123"):
- Hash of "user_123" % 100 = 23
- 23 < 10? No
- Sees: Old checkout flow

**User B** (userId: "user_456"):
- Hash of "user_456" % 100 = 7
- 7 < 10? Yes
- Sees: New checkout flow (with blue banner)

**Expected vs. Actual improvement**:
- Before: All users see same version (can't test in production)
- After: 10% of users see new version (gradual rollout)
- Before: Need to redeploy to change rollout
- After: Can update percentage in config (still need redeploy for now)
- Before: No way to target specific users
- After: Can use userList strategy for internal testing

### Server-Side Feature Flag API

For dynamic flag updates without redeployment:

```typescript
// src/app/api/feature-flags/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerEnv } from '@/config/env';
import { featureFlags, updateFeatureFlag } from '@/config/features';

/**
 * GET /api/feature-flags
 * List all feature flags
 */
export async function GET(request: NextRequest) {
  // In production, verify admin authentication
  const env = getServerEnv();
  const apiKey = request.headers.get('x-api-key');
  
  if (apiKey !== env.API_SECRET_KEY) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  
  return NextResponse.json({
    flags: Object.values(featureFlags),
  });
}

/**
 * PATCH /api/feature-flags/[key]
 * Update a feature flag
 */
export async function PATCH(request: NextRequest) {
  const env = getServerEnv();
  const apiKey = request.headers.get('x-api-key');
  
  if (apiKey !== env.API_SECRET_KEY) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  
  try {
    const body = await request.json();
    const { key, strategy } = body;
    
    if (!key || !strategy) {
      return NextResponse.json(
        { error: 'Missing required fields' },
        { status: 400 }
      );
    }
    
    updateFeatureFlag(key, { strategy });
    
    return NextResponse.json({
      success: true,
      flag: featureFlags[key],
    });
    
  } catch (error) {
    return NextResponse.json(
      { error: 'Invalid request' },
      { status: 400 }
    );
  }
}
```

### Dynamic Flag Updates

Now you can update flags without redeploying:

```bash
# Increase rollout to 50%
curl -X PATCH https://your-app.com/api/feature-flags \
  -H "x-api-key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "new-checkout",
    "strategy": { "type": "percentage", "rollout": 50 }
  }'

# Enable for all users
curl -X PATCH https://your-app.com/api/feature-flags \
  -H "x-api-key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "new-checkout",
    "strategy": { "type": "boolean", "enabled": true }
  }'

# Emergency disable
curl -X PATCH https://your-app.com/api/feature-flags \
  -H "x-api-key: your_secret_key" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "new-checkout",
    "strategy": { "type": "boolean", "enabled": false }
  }'
```

### Feature Flag Admin UI

A simple admin interface for managing flags:

```tsx
// src/app/admin/feature-flags/page.tsx
'use client';

import { useState, useEffect } from 'react';
import type { FeatureFlag } from '@/config/features';

export default function FeatureFlagsAdmin() {
  const [flags, setFlags] = useState<FeatureFlag[]>([]);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    loadFlags();
  }, []);

  async function loadFlags() {
    try {
      const response = await fetch('/api/feature-flags', {
        headers: {
          'x-api-key': process.env.NEXT_PUBLIC_ADMIN_KEY || '',
        },
      });
      
      const data = await response.json();
      setFlags(data.flags);
    } catch (error) {
      console.error('Failed to load flags:', error);
    } finally {
      setIsLoading(false);
    }
  }

  async function updateFlag(key: string, strategy: any) {
    try {
      await fetch('/api/feature-flags', {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          'x-api-key': process.env.NEXT_PUBLIC_ADMIN_KEY || '',
        },
        body: JSON.stringify({ key, strategy }),
      });
      
      await loadFlags();
    } catch (error) {
      console.error('Failed to update flag:', error);
    }
  }

  if (isLoading) {
    return <div className="p-6">Loading...</div>;
  }

  return (
    <div className="max-w-6xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Feature Flags</h1>
      
      <div className="space-y-4">
        {flags.map((flag) => (
          <div key={flag.key} className="border rounded-lg p-4">
            <div className="flex items-start justify-between mb-2">
              <div>
                <h3 className="font-semibold text-lg">{flag.name}</h3>
                <p className="text-sm text-gray-600">{flag.description}</p>
                <p className="text-xs text-gray-500 mt-1">Key: {flag.key}</p>
              </div>
              
              <div className="text-right">
                <div className="text-sm text-gray-600">
                  Updated: {new Date(flag.updatedAt).toLocaleDateString()}
                </div>
              </div>
            </div>
            
            <div className="mt-4">
              <StrategyEditor
                flag={flag}
                onUpdate={(strategy) => updateFlag(flag.key, strategy)}
              />
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}

function StrategyEditor({ 
  flag, 
  onUpdate 
}: { 
  flag: FeatureFlag; 
  onUpdate: (strategy: any) => void;
}) {
  const { strategy } = flag;
  
  if (strategy.type === 'boolean') {
    return (
      <div className="flex items-center gap-4">
        <span className="text-sm font-medium">Status:</span>
        <button
          onClick={() => onUpdate({ type: 'boolean', enabled: !strategy.enabled })}
          className={`px-4 py-2 rounded ${
            strategy.enabled 
              ? 'bg-green-600 text-white' 
              : 'bg-gray-300 text-gray-700'
          }`}
        >
          {strategy.enabled ? 'Enabled' : 'Disabled'}
        </button>
      </div>
    );
  }
  
  if (strategy.type === 'percentage') {
    return (
      <div className="flex items-center gap-4">
        <span className="text-sm font-medium">Rollout:</span>
        <input
          type="range"
          min="0"
          max="100"
          value={strategy.rollout}
          onChange={(e) => onUpdate({ 
            type: 'percentage', 
            rollout: parseInt(e.target.value) 
          })}
          className="flex-1"
        />
        <span className="text-sm font-semibold w-12">{strategy.rollout}%</span>
      </div>
    );
  }
  
  return (
    <div className="text-sm text-gray-600">
      Strategy: {strategy.type}
    </div>
  );
}
```

### Common Failure Mode: Inconsistent Flag Evaluation

**Symptom**: User sees different versions on different page loads

**Browser Console**:
```
Warning: Feature flag "new-checkout" evaluated differently
  First render: true
  Second render: false
```

**Root cause**: Feature flag evaluated without stable user context

**Solution**: Always provide consistent user context:

```tsx
// ❌ Bad: No user context (random results)
const enabled = useFeatureFlag('new-checkout');

// ✅ Good: Stable user context
const { user } = useUser();
const enabled = useFeatureFlag('new-checkout', {
  userId: user?.id,
  userEmail: user?.email,
});
```

### When to Apply This Solution

**What it optimizes for**:
- Safe production testing (gradual rollouts)
- Instant feature toggles (no redeployment)
- A/B testing capability
- Emergency kill switches

**What it sacrifices**:
- Additional code complexity
- Flag management overhead
- Potential for flag debt (old flags not cleaned up)

**When to choose this approach**:
- Production applications with active development
- Features that need gradual rollout
- A/B testing requirements
- High-risk features that need kill switches

**When to avoid this approach**:
- Simple applications with infrequent releases
- Features that don't need gradual rollout
- Prototypes or MVPs

**Code characteristics**:
- Setup: ~200 lines for basic system
- Maintenance: Medium (need to clean up old flags)
- Performance: Minimal (evaluation is fast)

### Limitation Preview

This simple system works well for basic use cases, but has limitations:

- **No real-time updates**: Flags cached until page reload
- **No analytics**: Can't track flag performance
- **No audit log**: Can't see who changed what when
- **No advanced targeting**: Can't target by location, device, etc.

For advanced needs, consider services like LaunchDarkly, Split.io, or Flagsmith. But for most applications, this simple system is sufficient.

## Analytics integration

## The Analytics Problem

Your application is live. Users are clicking buttons, completing purchases, encountering errors. But you have no idea:

- Which features are actually being used
- Where users are getting stuck
- What errors are happening in production
- How performance varies across users

**Without analytics**, you're flying blind. Let's build a comprehensive analytics system.

### Phase 1: The Naive Analytics Implementation

Here's the common mistake - directly calling analytics APIs everywhere:

```tsx
// src/app/checkout/page.tsx
'use client';

import { useState } from 'react';

export default function CheckoutPage() {
  const [isProcessing, setIsProcessing] = useState(false);

  async function handleCheckout() {
    // Direct analytics call
    window.gtag?.('event', 'checkout_started', {
      value: 99.99,
      currency: 'USD',
    });

    setIsProcessing(true);

    try {
      const response = await fetch('/api/checkout', {
        method: 'POST',
        body: JSON.stringify({ amount: 99.99 }),
      });

      if (!response.ok) {
        throw new Error('Checkout failed');
      }

      // Another direct analytics call
      window.gtag?.('event', 'purchase', {
        transaction_id: 'txn_123',
        value: 99.99,
        currency: 'USD',
      });

    } catch (error) {
      // Yet another direct analytics call
      window.gtag?.('event', 'checkout_error', {
        error_message: error instanceof Error ? error.message : 'Unknown',
      });
    } finally {
      setIsProcessing(false);
    }
  }

  return (
    <button onClick={handleCheckout}>
      Complete Purchase
    </button>
  );
}
```

### The Problems with Direct Analytics Calls

1. **Vendor lock-in**: Switching from Google Analytics to another service requires changing code everywhere
2. **No type safety**: Easy to send wrong event names or properties
3. **Inconsistent tracking**: Different developers track events differently
4. **No testing**: Can't test analytics without actually sending events
5. **No debugging**: Hard to see what events are being sent
6. **Privacy concerns**: No easy way to respect user consent

Let's build a better system.

## Building a Robust Analytics System

### The Analytics Abstraction Layer

```typescript
// src/lib/analytics/types.ts
// Type-safe analytics event definitions

/**
 * Standard e-commerce events
 * Based on Google Analytics 4 recommended events
 */
export type AnalyticsEvent =
  // Page views
  | { name: 'page_view'; properties: { page_path: string; page_title: string } }
  
  // E-commerce events
  | { name: 'view_item'; properties: { item_id: string; item_name: string; value: number } }
  | { name: 'add_to_cart'; properties: { item_id: string; item_name: string; value: number } }
  | { name: 'begin_checkout'; properties: { value: number; currency: string; items: number } }
  | { name: 'purchase'; properties: { 
      transaction_id: string; 
      value: number; 
      currency: string;
      items: number;
    }}
  
  // User actions
  | { name: 'sign_up'; properties: { method: string } }
  | { name: 'login'; properties: { method: string } }
  | { name: 'search'; properties: { search_term: string } }
  
  // Errors
  | { name: 'error'; properties: { 
      error_message: string; 
      error_location: string;
      error_type: string;
    }}
  
  // Custom events
  | { name: 'feature_used'; properties: { feature_name: string; context?: string } };

/**
 * User properties for analytics
 */
export interface AnalyticsUser {
  id?: string;
  email?: string;
  plan?: 'free' | 'pro' | 'enterprise';
  signupDate?: string;
}

/**
 * Analytics provider interface
 * Implement this for each analytics service
 */
export interface AnalyticsProvider {
  name: string;
  initialize(): void;
  trackEvent(event: AnalyticsEvent): void;
  identifyUser(user: AnalyticsUser): void;
  reset(): void;
}
```

```typescript
// src/lib/analytics/providers/google-analytics.ts
// Google Analytics 4 provider implementation

import type { AnalyticsProvider, AnalyticsEvent, AnalyticsUser } from '../types';
import { clientEnv } from '@/config/env';

export class GoogleAnalyticsProvider implements AnalyticsProvider {
  name = 'Google Analytics';
  private measurementId: string;
  private isInitialized = false;

  constructor(measurementId: string) {
    this.measurementId = measurementId;
  }

  initialize(): void {
    if (this.isInitialized || typeof window === 'undefined') {
      return;
    }

    // Load Google Analytics script
    const script = document.createElement('script');
    script.src = `https://www.googletagmanager.com/gtag/js?id=${this.measurementId}`;
    script.async = true;
    document.head.appendChild(script);

    // Initialize gtag
    window.dataLayer = window.dataLayer || [];
    window.gtag = function gtag() {
      window.dataLayer.push(arguments);
    };
    window.gtag('js', new Date());
    window.gtag('config', this.measurementId, {
      send_page_view: false, // We'll handle page views manually
    });

    this.isInitialized = true;
  }

  trackEvent(event: AnalyticsEvent): void {
    if (!this.isInitialized || !window.gtag) {
      return;
    }

    window.gtag('event', event.name, event.properties);
  }

  identifyUser(user: AnalyticsUser): void {
    if (!this.isInitialized || !window.gtag) {
      return;
    }

    window.gtag('set', 'user_properties', {
      user_id: user.id,
      plan: user.plan,
      signup_date: user.signupDate,
    });
  }

  reset(): void {
    // Google Analytics doesn't have a built-in reset
    // In practice, you'd clear cookies and reload
  }
}

// Type augmentation for window.gtag
declare global {
  interface Window {
    dataLayer: any[];
    gtag: (...args: any[]) => void;
  }
}
```

```typescript
// src/lib/analytics/providers/console.ts
// Console provider for development/debugging

import type { AnalyticsProvider, AnalyticsEvent, AnalyticsUser } from '../types';

export class ConsoleAnalyticsProvider implements AnalyticsProvider {
  name = 'Console';

  initialize(): void {
    console.log('📊 Analytics initialized (Console provider)');
  }

  trackEvent(event: AnalyticsEvent): void {
    console.log('📊 Analytics Event:', {
      name: event.name,
      properties: event.properties,
      timestamp: new Date().toISOString(),
    });
  }

  identifyUser(user: AnalyticsUser): void {
    console.log('📊 Analytics User:', user);
  }

  reset(): void {
    console.log('📊 Analytics reset');
  }
}
```

### The Analytics Manager

```typescript
// src/lib/analytics/index.ts
// Central analytics manager

import type { AnalyticsProvider, AnalyticsEvent, AnalyticsUser } from './types';
import { GoogleAnalyticsProvider } from './providers/google-analytics';
import { ConsoleAnalyticsProvider } from './providers/console';
import { clientEnv, isProduction } from '@/config/env';

class AnalyticsManager {
  private providers: AnalyticsProvider[] = [];
  private isInitialized = false;
  private consentGiven = false;
  private eventQueue: AnalyticsEvent[] = [];

  /**
   * Initialize analytics with configured providers
   */
  initialize(): void {
    if (this.isInitialized) {
      return;
    }

    // Always use console provider in development
    if (!isProduction) {
      this.providers.push(new ConsoleAnalyticsProvider());
    }

    // Add Google Analytics in production (if configured)
    if (clientEnv.NEXT_PUBLIC_ANALYTICS_ID) {
      this.providers.push(
        new GoogleAnalyticsProvider(clientEnv.NEXT_PUBLIC_ANALYTICS_ID)
      );
    }

    // Initialize all providers
    this.providers.forEach(provider => {
      try {
        provider.initialize();
      } catch (error) {
        console.error(`Failed to initialize ${provider.name}:`, error);
      }
    });

    this.isInitialized = true;

    // Check for stored consent
    this.checkConsent();
  }

  /**
   * Check if user has given analytics consent
   */
  private checkConsent(): void {
    if (typeof window === 'undefined') {
      return;
    }

    const consent = localStorage.getItem('analytics_consent');
    if (consent === 'granted') {
      this.grantConsent();
    }
  }

  /**
   * Grant analytics consent and flush queued events
   */
  grantConsent(): void {
    this.consentGiven = true;
    localStorage.setItem('analytics_consent', 'granted');

    // Flush queued events
    this.eventQueue.forEach(event => this.trackEvent(event));
    this.eventQueue = [];
  }

  /**
   * Revoke analytics consent
   */
  revokeConsent(): void {
    this.consentGiven = false;
    localStorage.removeItem('analytics_consent');
    this.eventQueue = [];
    
    // Reset all providers
    this.providers.forEach(provider => {
      try {
        provider.reset();
      } catch (error) {
        console.error(`Failed to reset ${provider.name}:`, error);
      }
    });
  }

  /**
   * Track an analytics event
   */
  trackEvent(event: AnalyticsEvent): void {
    if (!this.isInitialized) {
      console.warn('Analytics not initialized');
      return;
    }

    // Queue events if consent not given (except in development)
    if (!this.consentGiven && isProduction) {
      this.eventQueue.push(event);
      return;
    }

    // Send to all providers
    this.providers.forEach(provider => {
      try {
        provider.trackEvent(event);
      } catch (error) {
        console.error(`Failed to track event with ${provider.name}:`, error);
      }
    });
  }

  /**
   * Identify the current user
   */
  identifyUser(user: AnalyticsUser): void {
    if (!this.isInitialized || (!this.consentGiven && isProduction)) {
      return;
    }

    this.providers.forEach(provider => {
      try {
        provider.identifyUser(user);
      } catch (error) {
        console.error(`Failed to identify user with ${provider.name}:`, error);
      }
    });
  }

  /**
   * Track a page view
   */
  trackPageView(path: string, title: string): void {
    this.trackEvent({
      name: 'page_view',
      properties: { page_path: path, page_title: title },
    });
  }
}

// Export singleton instance
export const analytics = new AnalyticsManager();

// Export types for consumers
export type { AnalyticsEvent, AnalyticsUser } from './types';
```

### Iteration 1: Type-Safe Analytics in Components

Now let's use our analytics system in the checkout:

```tsx
// src/app/checkout/page.tsx
'use client';

import { useState } from 'react';
import { analytics } from '@/lib/analytics';

export default function CheckoutPage() {
  const [isProcessing, setIsProcessing] = useState(false);

  async function handleCheckout() {
    // Type-safe analytics event
    analytics.trackEvent({
      name: 'begin_checkout',
      properties: {
        value: 99.99,
        currency: 'USD',
        items: 1,
      },
    });

    setIsProcessing(true);

    try {
      const response = await fetch('/api/checkout', {
        method: 'POST',
        body: JSON.stringify({ amount: 99.99 }),
      });

      if (!response.ok) {
        throw new Error('Checkout failed');
      }

      const data = await response.json();

      // Track successful purchase
      analytics.trackEvent({
        name: 'purchase',
        properties: {
          transaction_id: data.orderId,
          value: 99.99,
          currency: 'USD',
          items: 1,
        },
      });

    } catch (error) {
      // Track error
      analytics.trackEvent({
        name: 'error',
        properties: {
          error_message: error instanceof Error ? error.message : 'Unknown',
          error_location: 'checkout',
          error_type: 'checkout_failed',
        },
      });
    } finally {
      setIsProcessing(false);
    }
  }

  return (
    <button onClick={handleCheckout}>
      Complete Purchase
    </button>
  );
}
```

**Verification**: In development, check the console:

**Browser Console**:
```
📊 Analytics initialized (Console provider)
📊 Analytics Event: {
  name: 'begin_checkout',
  properties: { value: 99.99, currency: 'USD', items: 1 },
  timestamp: '2024-01-15T10:30:00.000Z'
}
📊 Analytics Event: {
  name: 'purchase',
  properties: { 
    transaction_id: 'ord_abc123', 
    value: 99.99, 
    currency: 'USD',
    items: 1
  },
  timestamp: '2024-01-15T10:30:02.000Z'
}
```

**Expected vs. Actual improvement**:
- Before: Direct gtag calls, no type safety
- After: Type-safe events, autocomplete in IDE
- Before: Vendor lock-in to Google Analytics
- After: Can switch providers without changing component code
- Before: No development visibility
- After: Console logging in development

### Analytics Initialization

Initialize analytics when your app loads:

```tsx
// src/app/layout.tsx
import { useEffect } from 'react';
import { analytics } from '@/lib/analytics';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  useEffect(() => {
    // Initialize analytics on mount
    analytics.initialize();
  }, []);

  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

### Automatic Page View Tracking

Track page views automatically with Next.js navigation:

```tsx
// src/components/AnalyticsPageView.tsx
'use client';

import { useEffect } from 'react';
import { usePathname } from 'next/navigation';
import { analytics } from '@/lib/analytics';

export function AnalyticsPageView() {
  const pathname = usePathname();

  useEffect(() => {
    if (pathname) {
      analytics.trackPageView(pathname, document.title);
    }
  }, [pathname]);

  return null;
}
```

```tsx
// src/app/layout.tsx
import { AnalyticsPageView } from '@/components/AnalyticsPageView';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <AnalyticsPageView />
        {children}
      </body>
    </html>
  );
}
```

### User Consent Management

Implement GDPR-compliant consent:

```tsx
// src/components/CookieConsent.tsx
'use client';

import { useState, useEffect } from 'react';
import { analytics } from '@/lib/analytics';

export function CookieConsent() {
  const [showBanner, setShowBanner] = useState(false);

  useEffect(() => {
    // Check if user has already made a choice
    const consent = localStorage.getItem('analytics_consent');
    if (!consent) {
      setShowBanner(true);
    }
  }, []);

  function handleAccept() {
    analytics.grantConsent();
    setShowBanner(false);
  }

  function handleDecline() {
    analytics.revokeConsent();
    setShowBanner(false);
  }

  if (!showBanner) {
    return null;
  }

  return (
    <div className="fixed bottom-0 left-0 right-0 bg-gray-900 text-white p-4 shadow-lg z-50">
      <div className="max-w-6xl mx-auto flex items-center justify-between gap-4">
        <p className="text-sm">
          We use cookies to improve your experience and analyze site usage. 
          By clicking "Accept", you consent to our use of cookies.
        </p>
        
        <div className="flex gap-2 shrink-0">
          <button
            onClick={handleDecline}
            className="px-4 py-2 text-sm border border-gray-600 rounded hover:bg-gray-800"
          >
            Decline
          </button>
          <button
            onClick={handleAccept}
            className="px-4 py-2 text-sm bg-blue-600 rounded hover:bg-blue-700"
          >
            Accept
          </button>
        </div>
      </div>
    </div>
  );
}
```

### Custom Analytics Hooks

Create reusable hooks for common tracking patterns:

```typescript
// src/hooks/useAnalytics.ts
import { useEffect, useCallback } from 'react';
import { analytics, type AnalyticsEvent } from '@/lib/analytics';

/**
 * Track an event when component mounts
 */
export function useTrackMount(event: AnalyticsEvent) {
  useEffect(() => {
    analytics.trackEvent(event);
  }, []); // eslint-disable-line react-hooks/exhaustive-deps
}

/**
 * Get a callback to track an event
 */
export function useTrackEvent() {
  return useCallback((event: AnalyticsEvent) => {
    analytics.trackEvent(event);
  }, []);
}

/**
 * Track feature usage
 */
export function useTrackFeature(featureName: string, context?: string) {
  const trackEvent = useTrackEvent();
  
  return useCallback(() => {
    trackEvent({
      name: 'feature_used',
      properties: { feature_name: featureName, context },
    });
  }, [trackEvent, featureName, context]);
}
```

```tsx
// Usage example
import { useTrackFeature } from '@/hooks/useAnalytics';

export function SearchBar() {
  const trackSearch = useTrackFeature('search_bar', 'header');

  function handleSearch(query: string) {
    trackSearch();
    // ... perform search
  }

  return <input onChange={(e) => handleSearch(e.target.value)} />;
}
```

### Common Failure Mode: Analytics Not Loading

**Symptom**: No analytics events in production

**Browser Console**:
```
(No analytics logs - silent failure)
```

**Network Tab**:
- No requests to Google Analytics
- No gtag.js script loaded

**Root cause**: Ad blocker or privacy extension blocking analytics

**Solution**: 
1. Accept that some users will block analytics (respect their choice)
2. Use server-side analytics for critical metrics
3. Don't rely on analytics for core functionality
4. Test with ad blockers enabled

### When to Apply This Solution

**What it optimizes for**:
- Type safety (catch errors at compile time)
- Vendor flexibility (easy to switch providers)
- Privacy compliance (consent management)
- Development experience (console logging)

**What it sacrifices**:
- Initial setup complexity
- Additional abstraction layer
- Slightly more code per event

**When to choose this approach**:
- Production applications
- Multiple analytics providers
- GDPR/privacy compliance required
- Team collaboration (consistent tracking)

**When to avoid this approach**:
- Simple prototypes
- Single analytics provider that won't change
- No privacy compliance requirements

**Code characteristics**:
- Setup: ~300 lines for complete system
- Maintenance: Low (add events as needed)
- Performance: Minimal overhead

## Monitoring and alerting

## The Monitoring Problem

Your application is in production. Everything seems fine. Then you check your error tracking service and discover:

- 500 users hit a critical error in the last hour
- Your API response time increased 10x
- Memory usage is climbing steadily
- A deployment broke authentication for mobile users

**Without monitoring**, you only learn about problems when users complain. Let's build a comprehensive monitoring system.

### Phase 1: The Silent Failure

Here's what happens without monitoring:

```tsx
// src/app/api/checkout/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    
    // This might fail silently
    const result = await processPayment(body);
    
    return NextResponse.json(result);
    
  } catch (error) {
    // Error logged to console (which you're not watching)
    console.error('Checkout error:', error);
    
    return NextResponse.json(
      { error: 'Checkout failed' },
      { status: 500 }
    );
  }
}

async function processPayment(data: any) {
  // Simulated payment processing
  // What if this throws an error?
  // What if it's slow?
  // What if it fails for specific users?
  throw new Error('Payment gateway timeout');
}
```

### The Production Failure

**User Experience**:
- Checkout button shows "Processing..."
- After 30 seconds, shows "Checkout failed"
- User tries again, same result
- User abandons cart

**Your Visibility**:
- No alert
- No notification
- No dashboard showing the problem
- You only find out when checking logs manually (if at all)

**Business Impact**:
- Lost revenue
- Frustrated users
- Damaged reputation
- No data to diagnose the issue

### Diagnostic Analysis: What We're Missing

**What we need to know**:
1. **Errors**: What errors are happening and how often?
2. **Performance**: How fast are our APIs responding?
3. **Availability**: Is the service up and accessible?
4. **User Impact**: How many users are affected?
5. **Context**: What were they doing when it failed?

**What we need to do**:
1. **Capture**: Collect error data with full context
2. **Aggregate**: Group similar errors together
3. **Alert**: Notify team when thresholds are exceeded
4. **Visualize**: Dashboard showing system health
5. **Debug**: Provide enough context to fix issues

## Building a Production Monitoring System

We'll integrate Sentry for error tracking and build custom performance monitoring.

### Setting Up Sentry

First, install and configure Sentry:

```bash
npm install @sentry/nextjs
npx @sentry/wizard@latest -i nextjs
```

```typescript
// sentry.client.config.ts
// Client-side Sentry configuration

import * as Sentry from '@sentry/nextjs';
import { clientEnv, isProduction } from '@/config/env';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  
  // Only send errors in production
  enabled: isProduction,
  
  // Set environment
  environment: clientEnv.NEXT_PUBLIC_ENVIRONMENT,
  
  // Sample rate for performance monitoring
  tracesSampleRate: isProduction ? 0.1 : 1.0, // 10% in prod, 100% in dev
  
  // Sample rate for session replay
  replaysSessionSampleRate: 0.1, // 10% of sessions
  replaysOnErrorSampleRate: 1.0, // 100% of sessions with errors
  
  // Integrations
  integrations: [
    new Sentry.BrowserTracing({
      // Track navigation performance
      tracePropagationTargets: ['localhost', /^https:\/\/yourapp\.com/],
    }),
    new Sentry.Replay({
      // Mask sensitive data
      maskAllText: true,
      blockAllMedia: true,
    }),
  ],
  
  // Filter out known noise
  beforeSend(event, hint) {
    // Don't send errors from browser extensions
    if (event.exception?.values?.[0]?.stacktrace?.frames?.some(
      frame => frame.filename?.includes('extension://')
    )) {
      return null;
    }
    
    return event;
  },
});
```

```typescript
// sentry.server.config.ts
// Server-side Sentry configuration

import * as Sentry from '@sentry/nextjs';
import { getServerEnv } from '@/config/env';

const env = getServerEnv();

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  
  enabled: env.NODE_ENV === 'production',
  environment: env.NODE_ENV,
  
  tracesSampleRate: 0.1,
  
  // Server-specific options
  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
  ],
});
```

### Iteration 1: Monitored API Routes

Add comprehensive error tracking to API routes:

```typescript
// src/app/api/checkout/route.ts
import { NextRequest, NextResponse } from 'next/server';
import * as Sentry from '@sentry/nextjs';

export async function POST(request: NextRequest) {
  // Start a Sentry transaction for performance monitoring
  const transaction = Sentry.startTransaction({
    op: 'http.server',
    name: 'POST /api/checkout',
  });

  try {
    const body = await request.json();
    
    // Add context to Sentry
    Sentry.setContext('checkout', {
      amount: body.amount,
      items: body.items?.length,
    });
    
    // Add user context if available
    const userId = request.headers.get('x-user-id');
    if (userId) {
      Sentry.setUser({ id: userId });
    }
    
    // Track payment processing span
    const paymentSpan = transaction.startChild({
      op: 'payment.process',
      description: 'Process payment',
    });
    
    const result = await processPayment(body);
    
    paymentSpan.finish();
    
    // Track successful checkout
    Sentry.addBreadcrumb({
      category: 'checkout',
      message: 'Checkout completed successfully',
      level: 'info',
      data: { orderId: result.orderId },
    });
    
    transaction.setStatus('ok');
    transaction.finish();
    
    return NextResponse.json(result);
    
  } catch (error) {
    // Capture error with full context
    Sentry.captureException(error, {
      tags: {
        endpoint: '/api/checkout',
        method: 'POST',
      },
      contexts: {
        request: {
          url: request.url,
          method: request.method,
          headers: Object.fromEntries(request.headers),
        },
      },
    });
    
    transaction.setStatus('internal_error');
    transaction.finish();
    
    // Log for local debugging
    console.error('Checkout error:', error);
    
    return NextResponse.json(
      { 
        error: 'Checkout failed',
        // Don't expose internal error details to client
        message: error instanceof Error ? error.message : 'Unknown error',
      },
      { status: 500 }
    );
  }
}

async function processPayment(data: any) {
  // Simulated payment processing with detailed error
  throw new Error('Payment gateway timeout after 30s');
}
```

**Verification**: When the error occurs, Sentry captures:

**Sentry Dashboard**:
```
Error: Payment gateway timeout after 30s
  at processPayment (route.ts:45)
  at POST (route.ts:28)

Context:
  checkout: { amount: 99.99, items: 1 }
  user: { id: "user_123" }
  request: { url: "/api/checkout", method: "POST" }

Breadcrumbs:
  [info] checkout: Checkout started
  [error] checkout: Payment gateway timeout

Tags:
  endpoint: /api/checkout
  method: POST
  environment: production
```

**Expected vs. Actual improvement**:
- Before: Silent failure, no visibility
- After: Immediate error notification with full context
- Before: No way to know how many users affected
- After: Sentry shows error frequency and user impact
- Before: No debugging context
- After: Full request context, user info, breadcrumbs

### Client-Side Error Boundaries with Sentry

Catch React errors and send to Sentry:

```tsx
// src/components/ErrorBoundary.tsx
'use client';

import React from 'react';
import * as Sentry from '@sentry/nextjs';

interface Props {
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

interface State {
  hasError: boolean;
  error?: Error;
}

export class ErrorBoundary extends React.Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    // Send to Sentry with React-specific context
    Sentry.captureException(error, {
      contexts: {
        react: {
          componentStack: errorInfo.componentStack,
        },
      },
    });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="min-h-screen flex items-center justify-center p-6">
          <div className="max-w-md w-full bg-red-50 border border-red-200 rounded-lg p-6">
            <h2 className="text-xl font-semibold text-red-900 mb-2">
              Something went wrong
            </h2>
            <p className="text-red-700 mb-4">
              We've been notified and are working on a fix.
            </p>
            <button
              onClick={() => window.location.reload()}
              className="bg-red-600 text-white px-4 py-2 rounded hover:bg-red-700"
            >
              Reload page
            </button>
          </div>
        </div>
      );
    }

    return this.props.children;
  }
}
```

```tsx
// src/app/layout.tsx
import { ErrorBoundary } from '@/components/ErrorBoundary';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <ErrorBoundary>
          {children}
        </ErrorBoundary>
      </body>
    </html>
  );
}
```

### Custom Performance Monitoring

Track custom performance metrics:

```typescript
// src/lib/monitoring/performance.ts
// Custom performance monitoring

import * as Sentry from '@sentry/nextjs';

/**
 * Track API call performance
 */
export async function trackApiCall<T>(
  name: string,
  fn: () => Promise<T>
): Promise<T> {
  const transaction = Sentry.startTransaction({
    op: 'api.call',
    name,
  });

  const startTime = performance.now();

  try {
    const result = await fn();
    
    const duration = performance.now() - startTime;
    
    // Track successful call
    Sentry.addBreadcrumb({
      category: 'api',
      message: `${name} completed`,
      level: 'info',
      data: { duration },
    });
    
    transaction.setStatus('ok');
    transaction.finish();
    
    return result;
    
  } catch (error) {
    const duration = performance.now() - startTime;
    
    // Track failed call
    Sentry.captureException(error, {
      tags: { api_call: name },
      contexts: {
        performance: { duration },
      },
    });
    
    transaction.setStatus('internal_error');
    transaction.finish();
    
    throw error;
  }
}

/**
 * Track component render performance
 */
export function trackRender(componentName: string) {
  const transaction = Sentry.startTransaction({
    op: 'react.render',
    name: componentName,
  });

  return {
    finish: () => transaction.finish(),
  };
}

/**
 * Track custom metrics
 */
export function trackMetric(name: string, value: number, unit: string = 'ms') {
  Sentry.addBreadcrumb({
    category: 'metric',
    message: `${name}: ${value}${unit}`,
    level: 'info',
    data: { name, value, unit },
  });
}
```

```tsx
// Usage in components
import { trackApiCall, trackMetric } from '@/lib/monitoring/performance';

export default function CheckoutPage() {
  async function handleCheckout() {
    const startTime = performance.now();
    
    try {
      // Track API call performance
      const result = await trackApiCall('checkout', async () => {
        return fetch('/api/checkout', {
          method: 'POST',
          body: JSON.stringify({ amount: 99.99 }),
        }).then(r => r.json());
      });
      
      // Track custom metric
      const totalTime = performance.now() - startTime;
      trackMetric('checkout_total_time', totalTime);
      
    } catch (error) {
      // Error already tracked by trackApiCall
    }
  }

  return <button onClick={handleCheckout}>Checkout</button>;
}
```

### Alerting Configuration

Set up alerts in Sentry for critical issues:

**Sentry Alert Rules** (configured in Sentry dashboard):

1. **High Error Rate**:
   - Condition: More than 50 errors in 1 hour
   - Action: Send email + Slack notification
   - Priority: High

2. **Critical API Failure**:
   - Condition: Any error in `/api/checkout`
   - Action: Send PagerDuty alert
   - Priority: Critical

3. **Performance Degradation**:
   - Condition: P95 response time > 2 seconds
   - Action: Send Slack notification
   - Priority: Medium

4. **New Error Type**:
   - Condition: First occurrence of new error
   - Action: Send email
   - Priority: Low

### Health Check Endpoint

Create a health check for uptime monitoring:

```typescript
// src/app/api/health/route.ts
import { NextResponse } from 'next/server';
import { getServerEnv } from '@/config/env';

export async function GET() {
  const env = getServerEnv();
  
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    environment: env.NODE_ENV,
    checks: {
      database: await checkDatabase(),
      redis: await checkRedis(),
      externalApi: await checkExternalApi(),
    },
  };
  
  // If any check fails, return 503
  const isHealthy = Object.values(health.checks).every(check => check.status === 'ok');
  
  return NextResponse.json(health, {
    status: isHealthy ? 200 : 503,
  });
}

async function checkDatabase() {
  try {
    // Attempt database query
    // await db.query('SELECT 1');
    return { status: 'ok', latency: 5 };
  } catch (error) {
    return { status: 'error', error: 'Database connection failed' };
  }
}

async function checkRedis() {
  try {
    // Attempt Redis ping
    // await redis.ping();
    return { status: 'ok', latency: 2 };
  } catch (error) {
    return { status: 'error', error: 'Redis connection failed' };
  }
}

async function checkExternalApi() {
  try {
    // Check external API availability
    const response = await fetch('https://api.stripe.com/v1/health', {
      signal: AbortSignal.timeout(5000),
    });
    return { status: response.ok ? 'ok' : 'degraded', latency: 100 };
  } catch (error) {
    return { status: 'error', error: 'External API unreachable' };
  }
}
```

### Uptime Monitoring

Use a service like UptimeRobot or Pingdom to monitor your health endpoint:

**Configuration**:
- URL: `https://your-app.com/api/health`
- Interval: Every 5 minutes
- Alert: If status code is not 200
- Notification: Email + SMS for critical alerts

### Common Failure Mode: Alert Fatigue

**Symptom**: Too many alerts, team starts ignoring them

**Root cause**: Alerts not properly tuned

**Solution**:
1. **Set appropriate thresholds**: Don't alert on every error
2. **Group similar errors**: One alert for 100 similar errors, not 100 alerts
3. **Use severity levels**: Critical vs. warning vs. info
4. **Implement rate limiting**: Max 1 alert per hour for same issue
5. **Regular review**: Adjust thresholds based on actual patterns

### When to Apply This Solution

**What it optimizes for**:
- Visibility into production issues
- Fast incident response
- Debugging context
- Performance tracking

**What it sacrifices**:
- Additional service costs (Sentry, uptime monitoring)
- Setup complexity
- Potential privacy concerns (error data collection)

**When to choose this approach**:
- Any production application
- Applications with paying customers
- Team-maintained applications
- High-availability requirements

**When to avoid this approach**:
- Simple prototypes
- Personal projects with no users
- Applications with no uptime requirements

**Code characteristics**:
- Setup: ~500 lines including configuration
- Maintenance: Low (mostly configuration)
- Performance: Minimal overhead (sampling)

## The deployment checklist

## The Complete Pre-Deployment Checklist

You've built your application. You've tested it locally. You're ready to deploy. But are you _really_ ready?

This checklist covers everything you need to verify before deploying to production. Each item includes verification steps and common failure modes.

### Phase 1: Environment & Configuration

#### ✅ Environment Variables

**Verify**:

```bash
# Run environment validation
npm run validate-env

# Expected output:
# 🔍 Validating environment configuration...
# ✅ Client environment valid
#    Environment: production
#    API URL: https://api.yourapp.com
# ✅ Server environment valid
#    Node environment: production
# ✅ Environment configuration validated successfully
```

**Common Failure**:
```
❌ Invalid client environment variables: {
  NEXT_PUBLIC_API_URL: ['Required']
}
```

**Fix**: Set missing environment variables in deployment platform (Vercel, AWS, etc.)

#### ✅ Build Success

**Verify**:

```bash
# Run production build locally
npm run build

# Expected output:
# ✓ Compiled successfully
# ✓ Linting and checking validity of types
# ✓ Collecting page data
# ✓ Generating static pages (10/10)
# ✓ Finalizing page optimization
```

**Common Failures**:

1. **TypeScript errors**:
```
Type error: Property 'name' does not exist on type 'User | undefined'
```
**Fix**: Add proper type guards or optional chaining

2. **Missing dependencies**:
```
Module not found: Can't resolve '@/lib/utils'
```
**Fix**: Check import paths and installed packages

3. **Environment variable access**:
```
ReferenceError: process is not defined
```
**Fix**: Use `NEXT_PUBLIC_` prefix for client-side variables

#### ✅ Bundle Size

**Verify**:

```bash
# Analyze bundle size
npm run build
npx @next/bundle-analyzer

# Check output:
# First Load JS shared by all: 85 kB
# ├ chunks/framework.js: 45 kB
# ├ chunks/main.js: 30 kB
# └ chunks/webpack.js: 10 kB
```

**Warning Signs**:
- First Load JS > 200 kB (slow initial load)
- Individual page > 100 kB (consider code splitting)
- Duplicate dependencies (check bundle analyzer)

**Fix**: Use dynamic imports, optimize images, remove unused dependencies

### Phase 2: Security

#### ✅ Secrets Not in Code

**Verify**:

```bash
# Search for potential secrets in code
git grep -i "api_key\|secret\|password\|token" src/

# Should return ONLY references to environment variables:
# src/config/env.ts:  STRIPE_SECRET_KEY: z.string()
# src/lib/api.ts:  apiKey: process.env.STRIPE_SECRET_KEY
```

**Common Failure**:
```
src/lib/stripe.ts:  const apiKey = 'sk_live_abc123...'
```

**Fix**: Move to environment variables immediately, rotate the exposed secret

#### ✅ CORS Configuration

**Verify**:

```typescript
// src/middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Verify CORS headers are set correctly
  const response = NextResponse.next();
  
  // Only allow your domains
  const allowedOrigins = [
    'https://yourapp.com',
    'https://www.yourapp.com',
  ];
  
  const origin = request.headers.get('origin');
  if (origin && allowedOrigins.includes(origin)) {
    response.headers.set('Access-Control-Allow-Origin', origin);
  }
  
  return response;
}
```

**Common Failure**: Allowing all origins (`*`) in production

**Fix**: Explicitly list allowed origins

#### ✅ Authentication & Authorization

**Verify**:
- [ ] Protected routes require authentication
- [ ] API routes verify user permissions
- [ ] Session tokens have expiration
- [ ] Refresh token rotation implemented
- [ ] CSRF protection enabled

**Test**:

```bash
# Try accessing protected route without auth
curl https://your-app.com/api/admin/users

# Expected: 401 Unauthorized
# Actual: 200 OK with data ← SECURITY ISSUE
```

### Phase 3: Performance

#### ✅ Core Web Vitals

**Verify**:

```bash
# Run Lighthouse audit
npx lighthouse https://your-app.com --view

# Target scores:
# Performance: > 90
# Accessibility: > 90
# Best Practices: > 90
# SEO: > 90

# Core Web Vitals:
# LCP (Largest Contentful Paint): < 2.5s
# FID (First Input Delay): < 100ms
# CLS (Cumulative Layout Shift): < 0.1
```

**Common Failures**:

1. **High LCP** (slow loading):
   - Unoptimized images
   - Blocking JavaScript
   - Slow server response

2. **High CLS** (layout shift):
   - Images without dimensions
   - Dynamic content insertion
   - Web fonts loading

**Fix**: Use `next/image`, add dimensions, optimize fonts

#### ✅ Image Optimization

**Verify**:

```tsx
// ✅ Good: Using next/image
import Image from 'next/image';

<Image
  src="/hero.jpg"
  alt="Hero image"
  width={1200}
  height={600}
  priority // For above-the-fold images
/>

// ❌ Bad: Regular img tag
<img src="/hero.jpg" alt="Hero image" />
```

**Check**:
- [ ] All images use `next/image`
- [ ] Images have explicit width/height
- [ ] Above-the-fold images have `priority`
- [ ] Images are in modern formats (WebP, AVIF)

#### ✅ Database Query Performance

**Verify**:

```typescript
// Add query logging in development
import { getServerEnv } from '@/config/env';

const env = getServerEnv();

if (env.NODE_ENV === 'development') {
  // Log slow queries
  db.$on('query', (e) => {
    if (e.duration > 100) {
      console.warn(`Slow query (${e.duration}ms):`, e.query);
    }
  });
}
```

**Warning Signs**:
- Queries taking > 100ms
- N+1 query problems
- Missing database indexes
- Full table scans

**Fix**: Add indexes, use query optimization, implement caching

### Phase 4: Monitoring & Observability

#### ✅ Error Tracking

**Verify**:
- [ ] Sentry (or similar) configured
- [ ] Error boundaries in place
- [ ] API routes capture exceptions
- [ ] Source maps uploaded for production

**Test**:

```tsx
// Trigger a test error
function TestErrorButton() {
  return (
    <button onClick={() => {
      throw new Error('Test error - please ignore');
    }}>
      Test Error Tracking
    </button>
  );
}
```

**Expected**: Error appears in Sentry dashboard within 1 minute

#### ✅ Analytics

**Verify**:
- [ ] Analytics initialized
- [ ] Page views tracked
- [ ] Key events tracked (signup, purchase, etc.)
- [ ] User consent implemented (GDPR)

**Test**:

```bash
# Check analytics in browser console (development)
# Should see: 📊 Analytics Event: { name: 'page_view', ... }

# Check production analytics dashboard
# Should see: Real-time users, page views, events
```

#### ✅ Logging

**Verify**:
- [ ] Structured logging implemented
- [ ] Log levels configured (error, warn, info, debug)
- [ ] Sensitive data not logged
- [ ] Logs aggregated (CloudWatch, Datadog, etc.)

**Test**:

```typescript
// Good logging example
logger.info('User checkout started', {
  userId: user.id,
  amount: 99.99,
  items: 3,
  // Don't log: credit card numbers, passwords, tokens
});

// Bad logging example
console.log('Checkout:', JSON.stringify(checkoutData)); // May contain sensitive data
```

### Phase 5: User Experience

#### ✅ Loading States

**Verify**:
- [ ] All async operations show loading state
- [ ] Skeleton screens for content loading
- [ ] Optimistic updates where appropriate
- [ ] Error states with retry options

**Test**: Throttle network to "Slow 3G" in DevTools

**Expected**: User sees loading indicators, not blank screens

#### ✅ Error Handling

**Verify**:
- [ ] User-friendly error messages
- [ ] Error boundaries catch React errors
- [ ] API errors handled gracefully
- [ ] Network errors show retry option

**Test**:

```bash
# Simulate API error
curl -X POST https://your-app.com/api/checkout \
  -H "Content-Type: application/json" \
  -d '{"invalid": "data"}'

# Expected: User sees friendly error message
# Not: Raw error stack trace
```

#### ✅ Accessibility

**Verify**:
- [ ] Keyboard navigation works
- [ ] Screen reader tested
- [ ] Color contrast meets WCAG AA
- [ ] Focus indicators visible
- [ ] ARIA labels on interactive elements

**Test**:

```bash
# Run accessibility audit
npx lighthouse https://your-app.com --only-categories=accessibility

# Target: Score > 90

# Manual test: Navigate entire app using only keyboard
# Tab, Enter, Space, Arrow keys should work
```

### Phase 6: SEO & Social

#### ✅ Meta Tags

**Verify**:

```tsx
// src/app/layout.tsx
import { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Your App Name',
  description: 'Your app description',
  openGraph: {
    title: 'Your App Name',
    description: 'Your app description',
    images: ['/og-image.jpg'],
  },
  twitter: {
    card: 'summary_large_image',
    title: 'Your App Name',
    description: 'Your app description',
    images: ['/og-image.jpg'],
  },
};
```

**Test**:

```bash
# Check meta tags
curl -s https://your-app.com | grep -i "meta"

# Test social sharing
# Use: https://www.opengraph.xyz/
# Or: https://cards-dev.twitter.com/validator
```

#### ✅ Sitemap & Robots.txt

**Verify**:

```typescript
// src/app/sitemap.ts
import { MetadataRoute } from 'next';

export default function sitemap(): MetadataRoute.Sitemap {
  return [
    {
      url: 'https://yourapp.com',
      lastModified: new Date(),
      changeFrequency: 'daily',
      priority: 1,
    },
    {
      url: 'https://yourapp.com/about',
      lastModified: new Date(),
      changeFrequency: 'monthly',
      priority: 0.8,
    },
    // ... more pages
  ];
}
```

```typescript
// src/app/robots.ts
import { MetadataRoute } from 'next';

export default function robots(): MetadataRoute.Robots {
  return {
    rules: {
      userAgent: '*',
      allow: '/',
      disallow: ['/admin/', '/api/'],
    },
    sitemap: 'https://yourapp.com/sitemap.xml',
  };
}
```

**Test**:
```
https://your-app.com/sitemap.xml
https://your-app.com/robots.txt
```

### Phase 7: Deployment Platform

#### ✅ Vercel Configuration

**Verify**:

```json
// vercel.json
{
  "buildCommand": "npm run build",
  "devCommand": "npm run dev",
  "installCommand": "npm install",
  "framework": "nextjs",
  "regions": ["iad1"], // Choose closest to users
  "env": {
    "NEXT_PUBLIC_API_URL": "https://api.yourapp.com"
  }
}
```

**Check**:
- [ ] Environment variables set in Vercel dashboard
- [ ] Production domain configured
- [ ] SSL certificate active
- [ ] Preview deployments enabled
- [ ] Build cache enabled

#### ✅ Database Migrations

**Verify**:
- [ ] Migration scripts tested
- [ ] Rollback plan documented
- [ ] Database backup created
- [ ] Migration runs before deployment

**Test**:

```bash
# Run migrations in staging first
npm run db:migrate

# Verify schema
npm run db:verify

# Create backup
npm run db:backup
```

### Phase 8: Post-Deployment

#### ✅ Smoke Tests

**Immediately after deployment**:

```bash
# 1. Health check
curl https://your-app.com/api/health
# Expected: {"status":"healthy"}

# 2. Homepage loads
curl -I https://your-app.com
# Expected: 200 OK

# 3. Authentication works
curl -X POST https://your-app.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"test123"}'
# Expected: 200 OK with token

# 4. Critical API endpoint
curl https://your-app.com/api/products
# Expected: 200 OK with data
```

#### ✅ Monitor for 1 Hour

**Watch**:
- [ ] Error rate in Sentry (should be < 1%)
- [ ] Response times (should be < 500ms p95)
- [ ] User analytics (users can complete key flows)
- [ ] Server logs (no unexpected errors)

**Alert Thresholds**:
- Error rate > 5%: Investigate immediately
- Response time > 2s: Check performance
- Zero traffic: Check DNS/routing

### The Final Checklist

Print this and check off before every production deployment:

**Environment & Configuration**
- [ ] Environment variables validated
- [ ] Production build succeeds
- [ ] Bundle size acceptable (< 200 KB first load)

**Security**
- [ ] No secrets in code
- [ ] CORS configured correctly
- [ ] Authentication tested
- [ ] Authorization verified

**Performance**
- [ ] Lighthouse score > 90
- [ ] Core Web Vitals pass
- [ ] Images optimized
- [ ] Database queries optimized

**Monitoring**
- [ ] Error tracking active
- [ ] Analytics configured
- [ ] Logging implemented
- [ ] Alerts configured

**User Experience**
- [ ] Loading states present
- [ ] Error handling graceful
- [ ] Accessibility tested
- [ ] Mobile responsive

**SEO & Social**
- [ ] Meta tags configured
- [ ] Sitemap generated
- [ ] Robots.txt present
- [ ] Social sharing tested

**Deployment**
- [ ] Platform configured
- [ ] Database migrations ready
- [ ] Rollback plan documented
- [ ] Team notified

**Post-Deployment**
- [ ] Smoke tests pass
- [ ] Monitoring active
- [ ] No critical errors
- [ ] Users can complete key flows

### Common Production Failures and Their Signatures

#### Failure: Environment Variable Missing

**Symptom**: App crashes immediately after deployment

**Browser Console**:
```
Error: Invalid client environment configuration
```

**Fix**: Add missing variable in deployment platform, redeploy

#### Failure: Database Connection

**Symptom**: All API requests return 500

**Server Logs**:
```
Error: connect ECONNREFUSED
  at TCPConnectWrap.afterConnect
```

**Fix**: Check database URL, verify network access, check credentials

#### Failure: Memory Leak

**Symptom**: App slows down over time, eventually crashes

**Monitoring**:
- Memory usage climbing steadily
- Response times increasing
- Eventually: Out of memory errors

**Fix**: Check for unclosed connections, event listeners, large object retention

#### Failure: Rate Limiting

**Symptom**: Some users can't access app

**Server Logs**:
```
429 Too Many Requests
```

**Fix**: Implement proper rate limiting, add user feedback, increase limits if legitimate

### The Professional React Developer's Mental Model

**Before Deployment**:
1. Test in production-like environment
2. Verify all checklist items
3. Have rollback plan ready
4. Schedule during low-traffic period

**During Deployment**:
1. Monitor error rates
2. Watch response times
3. Check user analytics
4. Be ready to rollback

**After Deployment**:
1. Verify smoke tests pass
2. Monitor for 1 hour minimum
3. Check user feedback
4. Document any issues

**If Something Goes Wrong**:
1. Don't panic
2. Check monitoring dashboards
3. Review recent changes
4. Rollback if critical
5. Fix and redeploy

Remember: Every production deployment is a learning opportunity. Document what went wrong, update your checklist, and improve your process.
