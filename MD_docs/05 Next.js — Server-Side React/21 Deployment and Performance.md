# Chapter 21: Deployment and Performance

## Deploying to Vercel (the path of least resistance)

## The Final Mile: From Localhost to Live

So far, our application has lived exclusively on our local machine. It's fast, it's responsive, and it works perfectly... for an audience of one. The ultimate goal of web development is to share our creations with the world. This chapter is about crossing that final mile: deploying our Next.js application and ensuring it performs exceptionally for every user, everywhere.

We'll begin with the simplest, most integrated deployment experience for Next.js: Vercel. Vercel is the company behind Next.js, and their platform is purpose-built to host Next.js applications, offering a seamless "it just works" experience.

### Phase 1: Establish the Reference Implementation

To explore deployment and performance, we need an application to work with. Let's build a simple "DevProfile Showcase" app. It will display a developer's profile picture, a short bio, and a list of projects fetched from a mock API.

This initial version will be functional but deeply flawed from a performance and security perspective. These flaws are intentional; they will serve as the problems we will diagnose and solve throughout this chapter.

**Project Structure**:

```
src/
├── app/
│   ├── api/
│   │   └── projects/
│   │       └── route.ts  <-- Mock API for projects
│   ├── globals.css
│   ├── layout.tsx
│   └── page.tsx        <-- Our reference component
└── public/
    └── avatar.jpg      <-- A large, unoptimized image
```

First, let's create the main page component. It fetches user data and projects, and displays them. Notice the intentional problems: a hardcoded API key and a standard `<img>` tag.

## Environment variables and secrets

## Iteration 1: Securing Our Secrets

Our application is live, but it has a critical security vulnerability. We've committed a secret API key directly into our source code.

### Current Limitation: Exposed API Key

Anyone with access to our GitHub repository can see our `secret-api-key-do-not-commit`. If this were a real key for a paid service (like a database, payment gateway, or AI API), this would be a catastrophic leak, potentially leading to abuse and financial loss.

### Failure Demonstration: The Evidence in Plain Sight

The failure isn't a runtime error; it's a security flaw.

**Diagnostic Analysis: Reading the Failure**

**Code Repository Evidence**:
-   File: `src/app/page.tsx`
-   Line: `const apiKey = 'secret-api-key-do-not-commit';`
-   File: `src/app/api/projects/route.ts`
-   Line: `if (authHeader !== 'Bearer secret-api-key-do-not-commit')`

**Let's parse this evidence**:

1.  **What the user experiences**: The site works perfectly. There is no visible error. This is what makes security flaws so insidious.

2.  **What the repository reveals**: The secret is stored in plain text and is part of the Git history. Even if we remove it now, it remains in past commits unless we rewrite the history.

3.  **Root cause identified**: We have mixed configuration/secrets (the API key) with application code.

4.  **Why the current approach can't solve this**: Code committed to a public repository is, by definition, public. We cannot store private information there.

5.  **What we need**: A mechanism to provide secret values to our application *at runtime* in the deployment environment, without ever storing them in the source code. This mechanism is **environment variables**.

### Technique Introduced: Environment Variables

Environment variables are values that exist outside your application's code, provided by the operating system or hosting environment (like Vercel). Next.js has built-in support for them via `.env` files for local development and a UI for production environments.

There are two types of environment variables in Next.js:

1.  **Server-Side**: Accessible only in server-side code (Server Components, API routes, `getStaticProps`, etc.). These are the default. Example: `DATABASE_URL`, `API_KEY`.
2.  **Client-Side (Public)**: Accessible in browser-facing code. These **must** be prefixed with `NEXT_PUBLIC_`. Example: `NEXT_PUBLIC_GOOGLE_ANALYTICS_ID`.

### Solution Implementation: Using Environment Variables

Let's fix our security flaw.

#### Step 1: Create `.env.local` for Local Development

Create a new file at the root of your project called `.env.local`. This file should be added to `.gitignore` immediately to prevent it from ever being committed.

```bash
# .gitignore
# ... other entries
.env.local
```

Now, add your secret to this file.

```text
# .env.local
PROJECTS_API_KEY="secret-api-key-do-not-commit"
```

#### Step 2: Update the Code

Now, we'll refactor our code to read from `process.env`.

**Before** (Iteration 0):

```tsx
// src/app/page.tsx (snippet)
async function getProjects() {
  // DANGER: Hardcoded secret API key!
  const apiKey = 'secret-api-key-do-not-commit'; 
  // ...
  const res = await fetch(projectsUrl, {
    headers: { 'Authorization': `Bearer ${apiKey}` },
    // ...
  });
  // ...
}
```

```typescript
// src/app/api/projects/route.ts (snippet)
export async function GET(request: Request) {
  // ...
  if (authHeader !== 'Bearer secret-api-key-do-not-commit') {
    return NextResponse.json({ message: 'Unauthorized' }, { status: 401 });
  }
  // ...
}
```

**After** (Iteration 1):

```tsx
// src/app/page.tsx (snippet)
async function getProjects() {
  // ✅ Reading from environment variable
  const apiKey = process.env.PROJECTS_API_KEY; 
  // ...
  const res = await fetch(projectsUrl, {
    headers: { 'Authorization': `Bearer ${apiKey}` },
    // ...
  });
  // ...
}
```

```typescript
// src/app/api/projects/route.ts (snippet)
export async function GET(request: Request) {
  const authHeader = request.headers.get('Authorization');
  const apiKey = process.env.PROJECTS_API_KEY;

  if (authHeader !== `Bearer ${apiKey}`) {
    return NextResponse.json({ message: 'Unauthorized' }, { status: 401 });
  }

  return NextResponse.json(projects);
}
```

After making these changes, you must **restart your local development server** for Next.js to pick up the new variables from `.env.local`.

#### Step 3: Add Environment Variables to Vercel

Our code is fixed, but if we deploy it now, it will fail. Vercel doesn't have our `.env.local` file (because it's in `.gitignore`), so `process.env.PROJECTS_API_KEY` will be `undefined`.

1.  Go to your project's dashboard on Vercel.
2.  Click the "Settings" tab.
3.  In the side menu, click "Environment Variables".
4.  Add a new variable:
    -   **Name**: `PROJECTS_API_KEY`
    -   **Value**: `secret-api-key-do-not-commit`
5.  Ensure it's set for "Production", "Preview", and "Development" environments.
6.  Click "Save".

#### Step 4: Redeploy

Commit your code changes and push them to GitHub.

```bash
git add .
git commit -m "feat: Use environment variables for API key"
git push
```

Vercel will automatically detect the push and start a new deployment. This new build will have access to the environment variable you just configured.

### Verification

Visit your deployed URL. The site should function exactly as before. The project list loads correctly.

**Expected vs. Actual Improvement**:
-   **Expected**: The application continues to function, but the secret API key is no longer visible in the source code.
-   **Actual**: The application works, and checking the source code on GitHub confirms the key is gone, replaced by `process.env.PROJECTS_API_KEY`. Our application is now secure.

### Limitation Preview

Our app is secure, but it's slow. The large profile picture takes a long time to load, causing a poor user experience and hurting our performance scores. Next, we'll tackle image optimization.

## Image optimization with next/image

## Iteration 2: Taming the Image Beast

Our "DevProfile Showcase" app is secure, but it's not performant. The most glaring issue is the large, unoptimized avatar image. On a slow connection, users will see a blank space for seconds before the image appears, and the layout might shift once it does.

### Current Limitation: Unoptimized Images

We are using a standard `<img>` tag to serve a high-resolution JPEG file directly from our `/public` folder. This is inefficient for several reasons:
*   **File Size**: The browser is forced to download the full, multi-megabyte image, even if it's only displayed in a 150x150 pixel space.
*   **Format**: We're using JPEG, but modern formats like WebP or AVIF offer better compression at higher quality.
*   **Layout Shift**: Without explicit sizing placeholders, the page layout can reflow as the image loads, creating a jarring user experience. This is measured by the Core Web Vital metric **Cumulative Layout Shift (CLS)**.
*   **Loading Strategy**: The image is loaded immediately, even if it's off-screen, which can slow down the initial page render.

### Failure Demonstration: The Performance Report Card

Let's use Chrome DevTools to diagnose the problem on our live Vercel deployment.

1.  Open your deployed site in an incognito window.
2.  Open DevTools (`Cmd+Opt+I` or `Ctrl+Shift+I`).
3.  Go to the **Lighthouse** tab.
4.  Select "Performance" and "Desktop" (or "Mobile" for a stricter test).
5.  Click "Analyze page load".

**Diagnostic Analysis: Reading the Failure**

**Lighthouse Report Evidence**:
```
Performance: 65 (example score)

Opportunities:
- Serve images in next-gen formats: Potential savings of 1.8 MB
- Properly size images: Potential savings of 1.5 MB
- Defer offscreen images

Diagnostics:
- Largest Contentful Paint element: <img> (4.2s)
- Avoid large layout shifts: Cumulative Layout Shift score of 0.15
```

**Browser DevTools - Network Tab Analysis**:
-   Filter: Img
-   Observation: `avatar.jpg` is 2.1MB.
-   Timing: Takes 3.5s to load on a "Fast 3G" connection.
-   Waterfall: The image download blocks or delays other resources.

**Let's parse this evidence**:

1.  **What the user experiences**: The page loads, but the spot for the avatar is empty for several seconds. When the image finally appears, the text below it jumps down.

2.  **What the Lighthouse report reveals**: Our performance score is poor. It explicitly tells us to use better formats and resize the image. Our **Largest Contentful Paint (LCP)** is high because the largest element (the avatar) takes a long time to load. Our **Cumulative Layout Shift (CLS)** is also poor because of the layout jump.

3.  **What the Network tab shows**: The raw data confirms the diagnosis. We are sending a massive 2.1MB file when a file of a few kilobytes would suffice.

4.  **Root cause identified**: We are serving a raw, unoptimized image asset without any modern performance considerations.

5.  **What we need**: A component that automates image optimization: resizing, format conversion, lazy loading, and preventing layout shift.

### Technique Introduced: The `next/image` Component

Next.js provides a powerful, built-in solution: the `<Image>` component from `next/image`. It's a drop-in replacement for the `<img>` tag that solves all our problems automatically:

*   **Resizing**: Serves smaller, appropriately-sized images based on the device's viewport.
*   **Optimization**: Automatically converts images to modern, efficient formats like WebP on-the-fly.
*   **CLS Prevention**: Requires `width` and `height` props to automatically reserve space for the image before it loads, preventing layout shift.
*   **Lazy Loading**: Images are loaded by default only when they enter the viewport. You can mark critical images (like our avatar) for priority loading.

### Solution Implementation: Replacing `<img>` with `<Image>`

The change is minimal but impactful.

**Before** (Iteration 1):

```tsx
// src/app/page.tsx (snippet)
import { promises as fs } from 'fs';
import path from 'path';

// ...

export default async function HomePage() {
  const user = await getUserData();
  // ...
  return (
    <main className="container">
      <header className="profile-header">
        {/* PROBLEM: Using a standard, unoptimized img tag */}
        <img 
          src={user.avatarUrl} 
          alt={`${user.name}'s avatar`}
          width={150} 
          height={150}
          className="avatar"
        />
        {/* ... */}
      </header>
      {/* ... */}
    </main>
  );
}
```

**After** (Iteration 2):

```tsx
// src/app/page.tsx (snippet)
import Image from 'next/image'; // ← Import the component
import { promises as fs } from 'fs';
import path from 'path';

// ...

export default async function HomePage() {
  const user = await getUserData();
  // ...
  return (
    <main className="container">
      <header className="profile-header">
        {/* ✅ SOLUTION: Using the next/image component */}
        <Image
          src={user.avatarUrl}
          alt={`${user.name}'s avatar`}
          width={150}
          height={150}
          className="avatar"
          priority // ← Tell Next.js to load this critical image ASAP
        />
        {/* ... */}
      </header>
      {/* ... */}
    </main>
  );
}
```

We've made two changes:
1.  Imported `Image` from `next/image` and replaced the `<img>` tag.
2.  Added the `priority` prop. Since this avatar is "above the fold" (visible on initial load), this tells Next.js to preload it, which improves our LCP score.

Commit and push this change. Vercel will redeploy automatically.

### Verification

Let's re-run our analysis on the newly deployed site.

**Lighthouse Report Evidence (After)**:
```
Performance: 98 (example score)

Opportunities:
- (All image-related opportunities are gone)

Diagnostics:
- Largest Contentful Paint element: <Image> (1.1s)
- Avoid large layout shifts: Cumulative Layout Shift score of 0
```

**Browser DevTools - Network Tab Analysis (After)**:
-   Filter: Img
-   Observation: A request is made for `/_next/image?url=%2Favatar.jpg&w=384&q=75`. The response is a `.webp` file.
-   File Size: 35KB (down from 2.1MB!)
-   Timing: Loads in 250ms on a "Fast 3G" connection.

**Expected vs. Actual Improvement**:
-   **Expected**: The image should load faster and the performance scores should improve.
-   **Actual**: The improvements are dramatic. The image is now served as a tiny, next-gen WebP file. Our LCP is 4x faster, our CLS is eliminated, and our overall performance score is in the green. The user experience is significantly better.

### Limitation Preview

Our images are optimized, but what about our code? Large JavaScript bundles can make a site feel sluggish and unresponsive, even if it looks like it loaded quickly. Next, we'll analyze our JavaScript bundle and trim the fat.

## Bundle analysis and Core Web Vitals

## Iteration 3: Analyzing the JavaScript Bundle

Our app feels faster now that the images are optimized. However, another major factor in web performance is the size of the JavaScript bundle that a user must download, parse, and execute. Large bundles can lead to long interaction delays, measured by the Core Web Vital **Total Blocking Time (TBT)** or its field equivalent, **First Input Delay (FID)**.

### Current Limitation: Bloated Dependencies

To demonstrate this, let's introduce a notoriously large library into our project: `moment.js`. It's a powerful date/time library, but it's large and not tree-shakeable, making it a common cause of bundle bloat.

Let's add a "Member Since" date to our profile page using `moment`.

```bash
npm install moment
```

```tsx
// src/app/page.tsx (updated)
import Image from 'next/image';
import { promises as fs } from 'fs';
import path from 'path';
import moment from 'moment'; // ← PROBLEM: Importing a large library

async function getUserData() {
  return {
    name: 'Alex Doe',
    bio: 'A full-stack developer...',
    avatarUrl: '/avatar.jpg',
    joinDate: '2021-08-15T12:00:00Z', // ← Add join date
  };
}
// ... (getProjects remains the same)

export default async function HomePage() {
  const user = await getUserData();
  const projects = await getProjects();

  // Format the date using moment.js
  const memberSince = moment(user.joinDate).format('MMMM YYYY');

  return (
    <main className="container">
      <header className="profile-header">
        <Image
          src={user.avatarUrl}
          alt={`${user.name}'s avatar`}
          width={150}
          height={150}
          className="avatar"
          priority
        />
        <div>
          <h1>{user.name}</h1>
          <p>Member since {memberSince}</p> {/* ← Display formatted date */}
          <p>{user.bio}</p>
        </div>
      </header>
      {/* ... */}
    </main>
  );
}
```

Commit and push this change. The site will still work, but we've unknowingly introduced a performance bottleneck.

### Failure Demonstration: The Invisible Weight

The "failure" here is subtle. The site might not feel dramatically slower on a fast developer machine, but users on average devices or slower networks will feel it.

**Diagnostic Analysis: Reading the Failure**

**Lighthouse Report Evidence**:
```
Performance: 85 (down from 98)

Opportunities:
- Reduce unused JavaScript
- Minimize main-thread work

Diagnostics:
- Total Blocking Time: 350ms (previously ~50ms)
```

**Let's parse this evidence**:

1.  **What the user experiences**: The page appears quickly, but if they try to click or interact with something immediately, there might be a noticeable delay. The page feels "janky".

2.  **What the Lighthouse report reveals**: Our TBT has skyrocketed. This metric measures how long the main thread was blocked, preventing user interaction. The report suggests we are shipping JavaScript that isn't being used.

3.  **Root cause identified**: We've added a large, monolithic library to our application for a very small task (date formatting). The entire library is being added to our JavaScript bundle.

4.  **What we need**: A tool to visualize what's inside our JavaScript bundle so we can identify the largest contributors and find alternatives.

### Technique Introduced: Bundle Analysis with `@next/bundle-analyzer`

Next.js can be configured to generate a visual report of your application's bundles. This allows you to see exactly which packages are taking up the most space.

#### Step 1: Install the Analyzer

```bash
npm install @next/bundle-analyzer --save-dev
```

#### Step 2: Configure `next.config.js`

Update your `next.config.js` file to enable the analyzer when a specific environment variable is set.

```javascript
// next.config.js

const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

/** @type {import('next').NextConfig} */
const nextConfig = {
  // Your existing config can go here
};

module.exports = withBundleAnalyzer(nextConfig);
```

#### Step 3: Run the Build with Analysis

Now, run your build command with the `ANALYZE` variable set to `true`.

```bash
ANALYZE=true npm run build
```

This will perform the standard Next.js build and then open two HTML files in your browser: `client.html` and `server.html`. We are most concerned with `client.html`, which shows the JavaScript sent to the browser.

You will see a treemap visualization. The largest rectangle is `moment.js`, confirming our diagnosis. It's taking up a significant portion of the bundle.

### Solution Implementation: Replacing the Heavy Dependency

The solution is to replace `moment.js` with a lighter, more modern alternative like `date-fns` or `dayjs`. These libraries are modular, so you only import the functions you need. Let's use `date-fns`.

```bash
npm uninstall moment
npm install date-fns
```

Now, update the code to use `date-fns`.

**Before** (Iteration 2):

```tsx
// src/app/page.tsx (snippet)
import moment from 'moment';

// ...
const memberSince = moment(user.joinDate).format('MMMM YYYY');
// ...
```

**After** (Iteration 3):

```tsx
// src/app/page.tsx (snippet)
import { format, parseISO } from 'date-fns'; // ← Import specific functions

// ...
// Format the date using date-fns
const memberSince = format(parseISO(user.joinDate), 'MMMM yyyy');
// ...
```

### Verification

Let's re-run the bundle analyzer.

```bash
ANALYZE=true npm run build
```

In the new `client.html` report, the giant `moment.js` block is gone, replaced by a tiny sliver representing the `format` and `parseISO` functions from `date-fns`.

**Performance Metrics**:
-   **Before**:
    -   Page Bundle Size (JS): ~250KB (gzipped)
    -   TBT: 350ms
-   **After**:
    -   Page Bundle Size (JS): ~80KB (gzipped) (68% reduction)
    -   TBT: 45ms (87% improvement)

Commit and push the changes. A new Lighthouse report on the deployed site will confirm the improved TBT score.

### Core Web Vitals: A Quick Review

Throughout these optimizations, we've directly improved all three Core Web Vitals:
1.  **Largest Contentful Paint (LCP)**: Improved by using `next/image` with the `priority` prop.
2.  **Cumulative Layout Shift (CLS)**: Eliminated by providing `width` and `height` to `next/image`, which prevents content from jumping.
3.  **First Input Delay (FID) / Total Blocking Time (TBT)**: Improved by reducing the JavaScript bundle size, which frees up the main thread for user interaction.

## Edge runtime vs. Node.js runtime

## Iteration 4: Choosing the Right Runtime

Our application's frontend is now highly optimized. The final frontier for performance is the backend: the code that runs on the server to fetch data and render pages. In Next.js, this code can run in one of two environments: the **Node.js runtime** or the **Edge runtime**.

### Current State Recap

By default, all our server-side code (Server Components, API routes) runs in the Node.js runtime. This is a robust, full-featured environment that is hosted in a single region (e.g., `us-east-1`). When a user from Japan requests our page, their request must travel all the way to the US East server and back. This travel time is called **latency**.

### New Scenario: Minimizing Global Latency

What if we want our site to feel just as fast for a user in Tokyo as it does for a user in New York? We need to reduce latency by running our code closer to the user. This is precisely what the Edge runtime is for.

### Technique Introduced: The Edge Runtime

The Edge runtime is a lightweight JavaScript execution environment based on V8 isolates (the same technology behind Chrome). It's designed to be deployed globally across a content delivery network (CDN).

When you deploy a function to the Edge:
*   Your code is replicated in dozens of locations around the world.
*   When a user makes a request, it's automatically routed to the nearest physical server.
*   This dramatically reduces latency, resulting in faster data fetching and page loads for a global audience.

**Trade-offs**: The Edge runtime is not a full Node.js environment. It's a subset of web APIs and does not support native Node.js APIs like `fs` (filesystem access) or many database drivers that rely on them.

### Solution Implementation: Opting into the Edge Runtime

Let's convert our mock API route to run on the Edge. The change is a single line of code.

**Before** (Iteration 3 - Node.js Runtime):

```typescript
// src/app/api/projects/route.ts

import { NextResponse } from 'next/server';

// No runtime specified, defaults to 'nodejs'

const projects = [
  // ...
];

export async function GET(request: Request) {
  const authHeader = request.headers.get('Authorization');
  const apiKey = process.env.PROJECTS_API_KEY;

  if (authHeader !== `Bearer ${apiKey}`) {
    return NextResponse.json({ message: 'Unauthorized' }, { status: 401 });
  }

  return NextResponse.json(projects);
}
```

**After** (Iteration 4 - Edge Runtime):

```typescript
// src/app/api/projects/route.ts

import { NextResponse } from 'next/server';

export const runtime = 'edge'; // ← Opt into the Edge runtime

const projects = [
  // ...
];

export async function GET(request: Request) {
  const authHeader = request.headers.get('Authorization');
  const apiKey = process.env.PROJECTS_API_KEY;

  if (authHeader !== `Bearer ${apiKey}`) {
    return NextResponse.json({ message: 'Unauthorized' }, { status: 401 });
  }

  return NextResponse.json(projects);
}
```

By adding `export const runtime = 'edge'`, we've instructed Next.js and Vercel to deploy this API route as a globally distributed Edge Function.

### Verification

The functional behavior of the API will not change. The improvement is in non-functional performance (latency).

On Vercel, you can verify this:
1.  Go to your project dashboard and click the "Functions" tab.
2.  You will see the `/api/projects` function listed.
3.  The "Runtime" column will now say "Edge" instead of the region name (e.g., "IAD1" for Washington, D.C.).

This confirms the function is deployed globally. A user in Europe will now hit a server in Frankfurt or London instead of one in the US, receiving their data much faster.

### When to Apply This Solution: A Decision Framework

Choosing a runtime is a key architectural decision.

| Feature / Use Case                  | Choose Edge Runtime                                                              | Choose Node.js Runtime                                                              |
| ----------------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| **Primary Goal**                    | Lowest possible latency for a global audience.                                   | Maximum compatibility and computational power.                                      |
| **Typical Tasks**                   | API routes, middleware, personalization, A/B testing, streaming responses.       | Complex data processing, database connections (with traditional drivers), file system access. |
| **API Support**                     | Subset of Web APIs (`fetch`, `Request`, `Response`). No native Node.js APIs.     | Full Node.js API support (`fs`, `path`, `crypto`, etc.).                            |
| **Cold Starts**                     | Extremely fast (near zero).                                                      | Can be slower (hundreds of milliseconds).                                           |
| **Database Connections**            | Requires a database provider with a modern, HTTP-based driver (e.g., Neon, PlanetScale). | Compatible with any database that has a Node.js driver.                             |
| **Example**                         | A function that fetches user data from a serverless KV store and returns it.     | A function that generates a PDF report using a library that writes to a temp file.  |

For our `HomePage` Server Component, which reads from the filesystem (`fs`) in our simulation, it *must* run in the Node.js runtime. Our API route, however, is a perfect candidate for the Edge as it's simple and benefits greatly from low latency.

## Synthesis: The Complete Journey

## The Journey: From Problem to Solution

We took a simple, functional application and systematically hardened it for production. Each step addressed a specific failure mode, applying a targeted technique to improve security, performance, and user experience.

| Iteration | Failure Mode                               | Technique Applied                   | Result                                                              | Performance Impact                                     |
| :-------- | :----------------------------------------- | :---------------------------------- | :------------------------------------------------------------------ | :----------------------------------------------------- |
| 0         | No live deployment; code is local only.    | Vercel Git Integration              | Application is live on a public URL.                                | Baseline established.                                  |
| 1         | API key exposed in public source code.     | Environment Variables               | Secrets are securely managed in the deployment environment.         | None (Security fix).                                   |
| 2         | Large, unoptimized images causing slow LCP and high CLS. | `next/image` Component              | Images are resized, optimized, and served in modern formats.        | LCP, CLS dramatically improved. Image size reduced by >95%. |
| 3         | Bloated JS bundle causing high TBT.        | `@next/bundle-analyzer`, Dependency Swap | Identified and replaced heavy library with a lightweight alternative. | TBT improved by >80%. JS bundle size reduced by >60%.  |
| 4         | High global latency for API requests.      | Edge Runtime                        | API route is deployed globally, reducing latency for all users.     | Time To First Byte (TTFB) for API calls is minimized.  |

### Final Implementation

Here is the final, production-ready code for our main page, incorporating all our improvements.
