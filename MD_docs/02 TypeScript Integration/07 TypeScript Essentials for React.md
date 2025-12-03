# Chapter 7: TypeScript Essentials for React

## Why TypeScript is worth the initial friction

## Why TypeScript is worth the initial friction

You've built React components. They work. You can see them in the browser, interact with them, and ship them to users. So why add TypeScript—a layer that seems to do nothing but complain about your perfectly functional code?

Because **working code** and **maintainable code** are not the same thing.

### The Invisible Cost of JavaScript

JavaScript's flexibility is its greatest strength and its most dangerous weakness. Consider this component from Chapter 6:

```tsx
function UserProfile({ user }) {
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

This code works perfectly—until it doesn't. What happens when:
- Someone passes `user={null}`?
- The API changes `name` to `fullName`?
- A new developer adds `<p>{user.phone}</p>` but the `phone` property doesn't exist?

In JavaScript, you discover these problems at runtime, in production, when a user reports a blank screen.

### The Reference Implementation: User Dashboard (Revisited)

Let's return to the User Dashboard from Chapters 2-6. Here's the final version we built, now in plain JavaScript:

```jsx
// UserDashboard.jsx - JavaScript version from Chapter 6
import { useState, useEffect } from 'react';

function UserDashboard({ userId }) {
  const [user, setUser] = useState(null);
  const [activities, setActivities] = useState([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function fetchUserData() {
      try {
        const userRes = await fetch(`/api/users/${userId}`);
        const userData = await userRes.json();
        setUser(userData);

        const activitiesRes = await fetch(`/api/users/${userId}/activities`);
        const activitiesData = await activitiesRes.json();
        setActivities(activitiesData);
      } catch (err) {
        setError(err.message);
      } finally {
        setIsLoading(false);
      }
    }

    fetchUserData();
  }, [userId]);

  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorMessage message={error} />;

  return (
    <div className="dashboard">
      <UserProfile user={user} />
      <ActivityFeed activities={activities} />
    </div>
  );
}

function UserProfile({ user }) {
  return (
    <div className="profile">
      <img src={user.avatar} alt={user.name} />
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <p>Member since {user.joinDate}</p>
    </div>
  );
}

function ActivityFeed({ activities }) {
  return (
    <div className="feed">
      <h3>Recent Activity</h3>
      {activities.map(activity => (
        <ActivityItem key={activity.id} activity={activity} />
      ))}
    </div>
  );
}

function ActivityItem({ activity }) {
  return (
    <div className="activity-item">
      <span className="timestamp">{activity.timestamp}</span>
      <span className="action">{activity.action}</span>
      <span className="target">{activity.target}</span>
    </div>
  );
}
```

This code works. It's been tested. It's in production. But it's a ticking time bomb.

### The Failure: Runtime Errors TypeScript Should Have Caught

Let's introduce a realistic scenario: The backend team updates the API response format. They change the user object structure:

**Old API response**:
```json
{
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "avatar": "/avatars/alice.jpg",
  "joinDate": "2023-01-15"
}
```

**New API response**:
```json
{
  "firstName": "Alice",
  "lastName": "Johnson",
  "email": "alice@example.com",
  "avatarUrl": "/avatars/alice.jpg",
  "memberSince": "2023-01-15"
}
```

The backend team updates their documentation. They send an email. But you're busy shipping features. The change goes live.

**What happens in the browser**:

```text
Browser Console:

Warning: Failed prop type: Invalid prop `user.name` of type `undefined` supplied to `UserProfile`, expected `string`.
    at UserProfile (UserProfile.jsx:45)
    at UserDashboard (UserDashboard.jsx:12)

Uncaught TypeError: Cannot read properties of undefined (reading 'name')
    at UserProfile (UserProfile.jsx:48:23)
    at renderWithHooks (react-dom.development.js:16305)
```

**Browser Behavior**:
- User sees the loading spinner
- Loading spinner disappears
- Screen shows partial content: avatar image broken, name is blank, email displays correctly
- "Member since undefined" appears
- Browser console fills with errors

**React DevTools Evidence**:
- `UserProfile` component selected
- Props: `{ user: { firstName: "Alice", lastName: "Johnson", email: "alice@example.com", avatarUrl: "/avatars/alice.jpg", memberSince: "2023-01-15" } }`
- The component is trying to access `user.name` (doesn't exist)
- The component is trying to access `user.avatar` (doesn't exist)
- The component is trying to access `user.joinDate` (doesn't exist)

### Diagnostic Analysis: Terminal Type Errors vs. Browser Runtime Errors

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: See their profile with name, avatar, and join date
   - Actual: Broken image, blank name field, "Member since undefined"

2. **What the console reveals**:
   - Key indicator: `Cannot read properties of undefined (reading 'name')`
   - Error location: `UserProfile.jsx:48` (the line with `{user.name}`)
   - The component received a `user` object, but it doesn't have the expected properties

3. **Root cause identified**: 
   The component expects properties that no longer exist in the API response. The contract between frontend and backend has broken.

4. **Why JavaScript can't prevent this**:
   JavaScript has no way to know what properties `user` should have. It happily accepts any object and only fails when you try to access a non-existent property at runtime.

5. **What we need**:
   A way to define the shape of data at development time, so we catch these mismatches before they reach production.

### The TypeScript Solution: Catching Errors at Compile Time

Now let's see what happens with TypeScript. First, we define the shape of our data:

```typescript
// types.ts
export interface User {
  name: string;
  email: string;
  avatar: string;
  joinDate: string;
}

export interface Activity {
  id: string;
  timestamp: string;
  action: string;
  target: string;
}
```

Now we type our components:

```tsx
// UserDashboard.tsx - TypeScript version
import { useState, useEffect } from 'react';
import { User, Activity } from './types';

interface UserDashboardProps {
  userId: string;
}

function UserDashboard({ userId }: UserDashboardProps) {
  const [user, setUser] = useState<User | null>(null);
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function fetchUserData() {
      try {
        const userRes = await fetch(`/api/users/${userId}`);
        const userData = await userRes.json();
        setUser(userData); // ← TypeScript will check this

        const activitiesRes = await fetch(`/api/users/${userId}/activities`);
        const activitiesData = await activitiesRes.json();
        setActivities(activitiesData); // ← And this
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error');
      } finally {
        setIsLoading(false);
      }
    }

    fetchUserData();
  }, [userId]);

  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorMessage message={error} />;
  if (!user) return null;

  return (
    <div className="dashboard">
      <UserProfile user={user} />
      <ActivityFeed activities={activities} />
    </div>
  );
}

interface UserProfileProps {
  user: User;
}

function UserProfile({ user }: UserProfileProps) {
  return (
    <div className="profile">
      <img src={user.avatar} alt={user.name} />
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <p>Member since {user.joinDate}</p>
    </div>
  );
}
```

Now when the API changes, **before you even run the code**, TypeScript tells you:

**Terminal Output**:

```bash
src/components/UserDashboard.tsx:48:23 - error TS2339: 
Property 'name' does not exist on type '{ firstName: string; lastName: string; email: string; avatarUrl: string; memberSince: string; }'.

48       <h2>{user.name}</h2>
                     ~~~~

src/components/UserDashboard.tsx:46:20 - error TS2339:
Property 'avatar' does not exist on type '{ firstName: string; lastName: string; email: string; avatarUrl: string; memberSince: string; }'.

46       <img src={user.avatar} alt={user.name} />
                        ~~~~~~

src/components/UserDashboard.tsx:50:28 - error TS2339:
Property 'joinDate' does not exist on type '{ firstName: string; lastName: string; email: string; avatarUrl: string; memberSince: string; }'.

50       <p>Member since {user.joinDate}</p>
                                ~~~~~~~~

Found 3 errors in the same file, starting at: src/components/UserDashboard.tsx:46
```

**The difference**:
- **JavaScript**: Errors appear in production, reported by users
- **TypeScript**: Errors appear in your editor, before you even save the file

### The Cost-Benefit Analysis

**Initial friction** (one-time cost):
- Learning type syntax: 2-4 hours
- Setting up TypeScript in a project: 15 minutes
- Adding types to existing code: 1-2 hours per 1000 lines

**Ongoing benefits** (every day):
- Catch bugs at compile time instead of runtime
- Autocomplete shows you exactly what properties exist
- Refactoring is safe—TypeScript tells you every place that breaks
- Documentation is built into the code (types are always up-to-date)
- Onboarding new developers is faster (types explain the codebase)

**Real-world impact**:
- A study by Airbnb found that 38% of bugs could have been prevented by TypeScript
- Microsoft reported a 15% reduction in bugs after adopting TypeScript
- Developer velocity increases after the initial learning curve

### The Mental Model Shift

JavaScript asks: "Does this code run?"
TypeScript asks: "Does this code make sense?"

JavaScript is optimistic: "I'll try to make this work."
TypeScript is skeptical: "Prove to me this is correct."

The friction you feel is TypeScript forcing you to think about edge cases:
- What if `user` is `null`?
- What if the API returns an error?
- What if someone passes the wrong type of data?

These aren't TypeScript being pedantic. These are real scenarios that will happen in production. TypeScript makes you handle them upfront.

### When TypeScript Isn't Worth It

TypeScript has costs. It's not always the right choice:

**Skip TypeScript if**:
- You're building a quick prototype (< 500 lines)
- You're the only developer and the project is small
- The project has a short lifespan (< 3 months)
- You're learning React for the first time (learn React first, add TypeScript later)

**Use TypeScript if**:
- Multiple developers will work on the code
- The project will be maintained for > 6 months
- You're building a library or shared component system
- You're integrating with external APIs
- You value long-term maintainability over short-term speed

### The Path Forward

In this chapter, we'll convert our User Dashboard to TypeScript. We'll do it incrementally, learning the essential patterns that cover 80% of real-world React development. By the end, you'll understand:

- How to type props, state, and events
- How to handle nullable values safely
- How to type API responses
- How to build generic, reusable components
- Which TypeScript features matter (and which you can ignore)

The goal isn't to become a TypeScript expert. The goal is to write React code that fails at compile time instead of runtime.

Let's begin.

## Setting up TypeScript in your React project

## Setting up TypeScript in your React project

Before we can add types to our dashboard, we need a TypeScript-enabled React project. There are two scenarios: starting fresh or converting an existing project.

### Starting Fresh: Create a New TypeScript React Project

If you're starting a new project, Vite makes this trivial:

```bash
# Create a new React + TypeScript project
npm create vite@latest my-app -- --template react-ts

# Navigate into the project
cd my-app

# Install dependencies
npm install

# Start the development server
npm run dev
```

That's it. Vite has configured everything:
- TypeScript compiler (`tsc`)
- Type definitions for React (`@types/react`)
- Type definitions for React DOM (`@types/react-dom`)
- A `tsconfig.json` with sensible defaults

**Project Structure**:

```text
my-app/
├── src/
│   ├── App.tsx          ← .tsx for components
│   ├── main.tsx         ← Entry point
│   └── vite-env.d.ts    ← Vite type definitions
├── tsconfig.json        ← TypeScript configuration
├── tsconfig.node.json   ← TypeScript config for Vite
├── package.json
└── vite.config.ts       ← Vite configuration
```

### Converting an Existing Project: Adding TypeScript to Our Dashboard

More commonly, you have an existing React project (like our User Dashboard from Chapters 2-6) and want to add TypeScript. Here's the step-by-step process.

**Step 1: Install TypeScript and type definitions**

```bash
# Install TypeScript
npm install --save-dev typescript

# Install React type definitions
npm install --save-dev @types/react @types/react-dom

# Install Node.js type definitions (for Vite)
npm install --save-dev @types/node
```

**Step 2: Create a `tsconfig.json`**

This file tells TypeScript how to compile your code. Create it in your project root:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

**Key settings explained**:

- `"strict": true` - Enables all strict type checking. This is what catches bugs.
- `"jsx": "react-jsx"` - Tells TypeScript to use React 17+ JSX transform (no need to import React)
- `"noEmit": true` - TypeScript only checks types; Vite handles the actual compilation
- `"moduleResolution": "bundler"` - Modern module resolution for Vite/bundlers

**Step 3: Create `tsconfig.node.json`**

This separate config is for Vite's configuration file:

```json
{
  "compilerOptions": {
    "composite": true,
    "skipLibCheck": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true
  },
  "include": ["vite.config.ts"]
}
```

**Step 4: Rename files from `.jsx` to `.tsx`**

TypeScript uses different file extensions:
- `.js` → `.ts` (TypeScript files without JSX)
- `.jsx` → `.tsx` (TypeScript files with JSX/React components)

Rename your component files:

```bash
# Rename all .jsx files to .tsx
mv src/App.jsx src/App.tsx
mv src/components/UserDashboard.jsx src/components/UserDashboard.tsx
mv src/components/UserProfile.jsx src/components/UserProfile.tsx
mv src/components/ActivityFeed.jsx src/components/ActivityFeed.tsx
```

**Step 5: Update `vite.config.js` to `vite.config.ts`**

Rename and update your Vite config:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
});
```

**Step 6: Start the development server**

Now run your dev server. TypeScript will immediately start checking your code:

```bash
npm run dev
```

### The Failure: TypeScript Errors Everywhere

When you first run TypeScript on existing JavaScript code, you'll see errors. Lots of them. This is expected. Let's look at what TypeScript complains about in our dashboard:

**Terminal Output**:

```bash
src/components/UserDashboard.tsx:5:29 - error TS7031: 
Binding element 'userId' implicitly has an 'any' type.

5 function UserDashboard({ userId }) {
                              ~~~~~~

src/components/UserDashboard.tsx:6:26 - error TS7034:
Variable 'user' implicitly has type 'any' in some locations where its type cannot be determined.

6   const [user, setUser] = useState(null);
                           ~

src/components/UserDashboard.tsx:7:36 - error TS7034:
Variable 'activities' implicitly has type 'any[]' in some locations where its type cannot be determined.

7   const [activities, setActivities] = useState([]);
                                       ~

src/components/UserProfile.tsx:3:25 - error TS7031:
Binding element 'user' implicitly has an 'any' type.

3 function UserProfile({ user }) {
                          ~~~~

Found 12 errors in 4 files.
```

### Diagnostic Analysis: Reading TypeScript's Cryptic Messages

**Let's parse this evidence**:

1. **"Binding element 'userId' implicitly has an 'any' type"**:
   - Translation: "You're destructuring `userId` from props, but I don't know what type it is"
   - TypeScript needs to know: Is it a string? A number? Could it be undefined?

2. **"Variable 'user' implicitly has type 'any'"**:
   - Translation: "You're calling `useState(null)`, so I know the initial value is `null`, but what type will it be after you call `setUser`?"
   - TypeScript needs to know: What shape does the `user` object have?

3. **"Variable 'activities' implicitly has type 'any[]'"**:
   - Translation: "You're calling `useState([])`, so I know it's an array, but an array of what?"
   - TypeScript needs to know: What properties does each activity object have?

**Root cause identified**: 
TypeScript's `strict` mode requires explicit types. JavaScript's implicit typing doesn't work here.

**Why this is actually good**:
These errors are forcing us to document our assumptions. What *is* the shape of a user? What *is* the shape of an activity? In JavaScript, these assumptions lived only in our heads. In TypeScript, they're explicit and checked.

### The Incremental Migration Strategy

Don't try to fix all errors at once. Use this strategy:

**Phase 1: Add `// @ts-nocheck` to silence errors temporarily**

```tsx
// UserDashboard.tsx
// @ts-nocheck  ← Tells TypeScript to skip this file

import { useState, useEffect } from 'react';

function UserDashboard({ userId }) {
  // ... rest of the code
}
```

This lets you migrate files one at a time without breaking your entire build.

**Phase 2: Convert one file at a time**

Start with the simplest components (leaf nodes with no dependencies), then work up to complex components.

**Phase 3: Remove `// @ts-nocheck` when a file is fully typed**

Once all errors in a file are fixed, remove the comment. TypeScript will now check that file.

### Essential TypeScript Configuration Settings

Let's understand the key `tsconfig.json` settings that affect React development:

**Type checking strictness**:

```json
{
  "compilerOptions": {
    "strict": true,                    // Enable all strict checks
    "noImplicitAny": true,             // Error on implicit 'any' types
    "strictNullChecks": true,          // null and undefined are distinct types
    "strictFunctionTypes": true,       // Strict checking of function types
    "strictBindCallApply": true,       // Strict checking of bind/call/apply
    "noUnusedLocals": true,            // Error on unused local variables
    "noUnusedParameters": true,        // Error on unused function parameters
    "noFallthroughCasesInSwitch": true // Error on switch fallthrough
  }
}
```

**Recommendation**: Keep `"strict": true`. It catches the most bugs. If you're migrating a large codebase, you can temporarily disable specific checks:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": false,  // Temporarily allow implicit 'any' during migration
    "strictNullChecks": false // Temporarily allow null/undefined confusion
  }
}
```

But re-enable them as soon as possible. They're the most valuable checks.

**Module resolution**:

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",  // Modern resolution for Vite/webpack
    "resolveJsonModule": true,      // Allow importing .json files
    "allowImportingTsExtensions": true // Allow .ts/.tsx in imports
  }
}
```

**JSX configuration**:

```json
{
  "compilerOptions": {
    "jsx": "react-jsx"  // React 17+ JSX transform (no need to import React)
  }
}
```

### Verifying Your Setup

Create a simple test component to verify TypeScript is working:

```tsx
// src/components/TypeScriptTest.tsx
interface Props {
  message: string;
  count: number;
}

function TypeScriptTest({ message, count }: Props) {
  return (
    <div>
      <p>{message}</p>
      <p>Count: {count}</p>
    </div>
  );
}

export default TypeScriptTest;
```

Now try to use it incorrectly:

```tsx
// src/App.tsx
import TypeScriptTest from './components/TypeScriptTest';

function App() {
  return (
    <div>
      {/* This should show an error */}
      <TypeScriptTest message="Hello" count="not a number" />
    </div>
  );
}
```

**Expected Terminal Output**:

```bash
src/App.tsx:7:44 - error TS2322: 
Type 'string' is not assignable to type 'number'.

7       <TypeScriptTest message="Hello" count="not a number" />
                                             ~~~~~~~~~~~~~~~
```

If you see this error, TypeScript is working correctly. Fix it:

```tsx
// src/App.tsx
<TypeScriptTest message="Hello" count={42} />
```

The error disappears. TypeScript is now protecting you from type mismatches.

### Editor Integration: Making TypeScript Useful

TypeScript's real power comes from editor integration. Install the TypeScript extension for your editor:

**VS Code** (recommended):
- TypeScript support is built-in
- Install "Error Lens" extension to see errors inline
- Install "Pretty TypeScript Errors" for readable error messages

**Key editor features**:

1. **Hover to see types**: Hover over any variable to see its inferred type
2. **Autocomplete**: Type `user.` and see all available properties
3. **Go to definition**: Cmd/Ctrl + Click on a type to see its definition
4. **Rename symbol**: Rename a variable and TypeScript updates all references
5. **Inline errors**: See type errors directly in your code, not just in the terminal

### Common Setup Issues and Solutions

**Issue 1: "Cannot find module 'react'"**

**Solution**: Install type definitions:

```bash
npm install --save-dev @types/react @types/react-dom
```

**Issue 2: "JSX element implicitly has type 'any'"**

**Solution**: Add `"jsx": "react-jsx"` to `tsconfig.json`:

```json
{
  "compilerOptions": {
    "jsx": "react-jsx"
  }
}
```

**Issue 3: "Cannot use JSX unless the '--jsx' flag is provided"**

**Solution**: Make sure your file has a `.tsx` extension, not `.ts`.

**Issue 4: Vite doesn't recognize `.tsx` files**

**Solution**: Update `vite.config.ts`:

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  resolve: {
    extensions: ['.tsx', '.ts', '.jsx', '.js']
  }
});
```

### The Setup Checklist

Before moving forward, verify:

- ✅ TypeScript installed (`npm list typescript`)
- ✅ React type definitions installed (`npm list @types/react`)
- ✅ `tsconfig.json` exists with `"strict": true`
- ✅ Component files renamed to `.tsx`
- ✅ Dev server runs without crashing (`npm run dev`)
- ✅ Editor shows inline type errors
- ✅ Hover over variables shows inferred types

If all checks pass, you're ready to start typing your components.

## Typing props, state, and events

## Typing props, state, and events

Now that TypeScript is set up, let's convert our User Dashboard to use proper types. We'll do this incrementally, starting with the simplest patterns and building up to complex scenarios.

### Iteration 1: Typing Component Props

The most common TypeScript pattern in React is typing component props. Let's start with the simplest component from our dashboard:

```tsx
// LoadingSpinner.tsx - Before TypeScript
function LoadingSpinner() {
  return (
    <div className="spinner">
      <div className="spinner-circle"></div>
    </div>
  );
}

export default LoadingSpinner;
```

This component has no props, so it needs no type annotation. TypeScript infers the return type automatically. But let's add an optional `size` prop:

```tsx
// LoadingSpinner.tsx - After TypeScript
interface LoadingSpinnerProps {
  size?: 'small' | 'medium' | 'large';
}

function LoadingSpinner({ size = 'medium' }: LoadingSpinnerProps) {
  return (
    <div className={`spinner spinner-${size}`}>
      <div className="spinner-circle"></div>
    </div>
  );
}

export default LoadingSpinner;
```

**What changed**:

1. **Interface definition**: `interface LoadingSpinnerProps` defines the shape of props
2. **Optional property**: `size?` means the prop is optional (can be omitted)
3. **Union type**: `'small' | 'medium' | 'large'` means `size` can only be one of these three strings
4. **Default value**: `size = 'medium'` provides a fallback when the prop is omitted
5. **Type annotation**: `: LoadingSpinnerProps` tells TypeScript what props this component accepts

**Why this matters**:

Try to use the component incorrectly:

```tsx
// This will show a TypeScript error
<LoadingSpinner size="huge" />
```

**Terminal Output**:

```bash
error TS2322: Type '"huge"' is not assignable to type '"small" | "medium" | "large" | undefined'.
```

TypeScript prevents you from passing invalid values. The error appears in your editor before you even save the file.

### Iteration 2: Typing State with useState

Now let's type the `UserDashboard` component's state. Here's the JavaScript version:

```jsx
// UserDashboard.jsx - JavaScript version
function UserDashboard({ userId }) {
  const [user, setUser] = useState(null);
  const [activities, setActivities] = useState([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);
  
  // ...
}
```

TypeScript complains about this code:

```bash
error TS7034: Variable 'user' implicitly has type 'any' in some locations where its type cannot be determined.
error TS7034: Variable 'activities' implicitly has type 'any[]' in some locations where its type cannot be determined.
```

**The problem**: TypeScript can't infer what type `user` will be after we call `setUser(userData)`. It only knows the initial value is `null`.

**The solution**: Explicitly provide a type parameter to `useState`:

```tsx
// types.ts - Define our data shapes
export interface User {
  id: string;
  name: string;
  email: string;
  avatar: string;
  joinDate: string;
}

export interface Activity {
  id: string;
  timestamp: string;
  action: string;
  target: string;
}
```

```tsx
// UserDashboard.tsx - TypeScript version
import { useState, useEffect } from 'react';
import { User, Activity } from './types';

interface UserDashboardProps {
  userId: string;
}

function UserDashboard({ userId }: UserDashboardProps) {
  const [user, setUser] = useState<User | null>(null);
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState<boolean>(true);
  const [error, setError] = useState<string | null>(null);
  
  // ...
}
```

**What changed**:

1. **Type parameter**: `useState<User | null>(null)` tells TypeScript:
   - Initial value is `null`
   - After calling `setUser`, the value will be a `User` object or `null`

2. **Union type**: `User | null` means the state can be either a `User` or `null`

3. **Array type**: `Activity[]` means an array of `Activity` objects

4. **Primitive types**: `boolean` and `string | null` are explicit (though TypeScript could infer these)

**Why this matters**:

Now TypeScript knows the shape of `user`. Try to access a non-existent property:

```tsx
// This will show a TypeScript error
<h1>{user.fullName}</h1>
```

**Terminal Output**:

```bash
error TS2339: Property 'fullName' does not exist on type 'User'.
```

TypeScript tells you exactly what properties exist on `User`. Your editor's autocomplete will show you: `id`, `name`, `email`, `avatar`, `joinDate`.

### The Failure: Accessing Nullable State

With our typed state, let's try to render the user's name:

```tsx
function UserDashboard({ userId }: UserDashboardProps) {
  const [user, setUser] = useState<User | null>(null);
  
  return (
    <div>
      <h1>{user.name}</h1>
    </div>
  );
}
```

**Terminal Output**:

```bash
error TS2531: Object is possibly 'null'.

5       <h1>{user.name}</h1>
              ~~~~
```

### Diagnostic Analysis: Reading TypeScript's Null Safety

**Let's parse this evidence**:

1. **What TypeScript is telling us**:
   - "Object is possibly 'null'" means `user` might be `null`
   - TypeScript knows we initialized `user` with `null`
   - TypeScript doesn't know if we've called `setUser` yet

2. **Why this is a real problem**:
   - If we render before the API call completes, `user` is `null`
   - Accessing `user.name` when `user` is `null` causes a runtime error
   - JavaScript would let this crash in production

3. **Root cause identified**:
   TypeScript's `strictNullChecks` prevents accessing properties on potentially null values.

4. **What we need**:
   A way to tell TypeScript "I've checked that `user` is not null before accessing its properties."

**The solution**: Null checks and type guards:

```tsx
function UserDashboard({ userId }: UserDashboardProps) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  
  if (isLoading) {
    return <LoadingSpinner />;
  }
  
  if (!user) {
    return <div>No user found</div>;
  }
  
  // TypeScript now knows user is not null here
  return (
    <div>
      <h1>{user.name}</h1>
    </div>
  );
}
```

**What changed**:

1. **Early returns**: We return early if `isLoading` or if `user` is null
2. **Type narrowing**: After the `if (!user)` check, TypeScript knows `user` is not null in the remaining code
3. **Safe access**: We can now access `user.name` without errors

This pattern is called **type narrowing** or **type guards**. TypeScript tracks control flow and narrows types based on your checks.

### Iteration 3: Typing Event Handlers

Let's add a search feature to our dashboard. First, the JavaScript version:

```jsx
// SearchBar.jsx - JavaScript version
function SearchBar({ onSearch }) {
  const [query, setQuery] = useState('');
  
  const handleSubmit = (e) => {
    e.preventDefault();
    onSearch(query);
  };
  
  const handleChange = (e) => {
    setQuery(e.target.value);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input 
        type="text" 
        value={query} 
        onChange={handleChange}
        placeholder="Search activities..."
      />
      <button type="submit">Search</button>
    </form>
  );
}
```

Now let's add TypeScript. The challenge is typing the event handlers:

```tsx
// SearchBar.tsx - TypeScript version
import { useState, FormEvent, ChangeEvent } from 'react';

interface SearchBarProps {
  onSearch: (query: string) => void;
}

function SearchBar({ onSearch }: SearchBarProps) {
  const [query, setQuery] = useState<string>('');
  
  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    onSearch(query);
  };
  
  const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input 
        type="text" 
        value={query} 
        onChange={handleChange}
        placeholder="Search activities..."
      />
      <button type="submit">Search</button>
    </form>
  );
}

export default SearchBar;
```

**What changed**:

1. **Import event types**: `FormEvent` and `ChangeEvent` from React
2. **Type the callback**: `onSearch: (query: string) => void` means:
   - `onSearch` is a function
   - It takes one parameter: `query` of type `string`
   - It returns `void` (nothing)

3. **Type form submit**: `FormEvent<HTMLFormElement>` is the type for form submission events
4. **Type input change**: `ChangeEvent<HTMLInputElement>` is the type for input change events

**Common React event types**:

- `MouseEvent<HTMLButtonElement>` - Button clicks
- `ChangeEvent<HTMLInputElement>` - Input changes
- `ChangeEvent<HTMLSelectElement>` - Select changes
- `FormEvent<HTMLFormElement>` - Form submissions
- `KeyboardEvent<HTMLInputElement>` - Keyboard events
- `FocusEvent<HTMLInputElement>` - Focus/blur events

**Pro tip**: Let TypeScript infer event types when possible:

```tsx
// Instead of explicitly typing the event
const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
  setQuery(e.target.value);
};

// You can let TypeScript infer it from the JSX
<input onChange={(e) => setQuery(e.target.value)} />
```

When you use an inline arrow function, TypeScript infers the event type from the element. This is often cleaner for simple handlers.

### Iteration 4: Typing the Complete UserDashboard

Now let's put it all together. Here's the complete TypeScript version of our dashboard:

```tsx
// types.ts
export interface User {
  id: string;
  name: string;
  email: string;
  avatar: string;
  joinDate: string;
}

export interface Activity {
  id: string;
  timestamp: string;
  action: string;
  target: string;
}
```

```tsx
// UserDashboard.tsx
import { useState, useEffect } from 'react';
import { User, Activity } from './types';
import UserProfile from './UserProfile';
import ActivityFeed from './ActivityFeed';
import LoadingSpinner from './LoadingSpinner';
import ErrorMessage from './ErrorMessage';

interface UserDashboardProps {
  userId: string;
}

function UserDashboard({ userId }: UserDashboardProps) {
  const [user, setUser] = useState<User | null>(null);
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState<boolean>(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function fetchUserData() {
      try {
        const userRes = await fetch(`/api/users/${userId}`);
        if (!userRes.ok) {
          throw new Error(`Failed to fetch user: ${userRes.status}`);
        }
        const userData: User = await userRes.json();
        setUser(userData);

        const activitiesRes = await fetch(`/api/users/${userId}/activities`);
        if (!activitiesRes.ok) {
          throw new Error(`Failed to fetch activities: ${activitiesRes.status}`);
        }
        const activitiesData: Activity[] = await activitiesRes.json();
        setActivities(activitiesData);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error');
      } finally {
        setIsLoading(false);
      }
    }

    fetchUserData();
  }, [userId]);

  if (isLoading) {
    return <LoadingSpinner size="large" />;
  }

  if (error) {
    return <ErrorMessage message={error} />;
  }

  if (!user) {
    return <div>No user found</div>;
  }

  return (
    <div className="dashboard">
      <UserProfile user={user} />
      <ActivityFeed activities={activities} />
    </div>
  );
}

export default UserDashboard;
```

```tsx
// UserProfile.tsx
import { User } from './types';

interface UserProfileProps {
  user: User;
}

function UserProfile({ user }: UserProfileProps) {
  return (
    <div className="profile">
      <img src={user.avatar} alt={user.name} />
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <p>Member since {user.joinDate}</p>
    </div>
  );
}

export default UserProfile;
```

```tsx
// ActivityFeed.tsx
import { Activity } from './types';
import ActivityItem from './ActivityItem';

interface ActivityFeedProps {
  activities: Activity[];
}

function ActivityFeed({ activities }: ActivityFeedProps) {
  if (activities.length === 0) {
    return <p>No recent activity</p>;
  }

  return (
    <div className="feed">
      <h3>Recent Activity</h3>
      {activities.map(activity => (
        <ActivityItem key={activity.id} activity={activity} />
      ))}
    </div>
  );
}

export default ActivityFeed;
```

```tsx
// ActivityItem.tsx
import { Activity } from './types';

interface ActivityItemProps {
  activity: Activity;
}

function ActivityItem({ activity }: ActivityItemProps) {
  return (
    <div className="activity-item">
      <span className="timestamp">{activity.timestamp}</span>
      <span className="action">{activity.action}</span>
      <span className="target">{activity.target}</span>
    </div>
  );
}

export default ActivityItem;
```

### Expected vs. Actual Improvement

**Before TypeScript**:
- Runtime errors when API changes
- No autocomplete for object properties
- Unclear what props components accept
- Refactoring is risky (might break things silently)

**After TypeScript**:
- Compile-time errors when API changes
- Full autocomplete for `user.` and `activity.`
- Clear prop contracts in every component
- Safe refactoring (TypeScript tells you what breaks)

**Verification**: Let's intentionally break something to see TypeScript catch it:

```tsx
// Try to pass wrong prop type
<UserProfile user={null} />
```

**Terminal Output**:

```bash
error TS2322: Type 'null' is not assignable to type 'User'.
```

TypeScript prevents the error before it reaches the browser.

### Common Patterns and Shortcuts

**Pattern 1: Optional props with defaults**

```tsx
interface ButtonProps {
  label: string;
  variant?: 'primary' | 'secondary';
  disabled?: boolean;
}

function Button({ label, variant = 'primary', disabled = false }: ButtonProps) {
  return (
    <button className={`btn btn-${variant}`} disabled={disabled}>
      {label}
    </button>
  );
}
```

**Pattern 2: Children prop**

```tsx
import { ReactNode } from 'react';

interface CardProps {
  title: string;
  children: ReactNode;
}

function Card({ title, children }: CardProps) {
  return (
    <div className="card">
      <h3>{title}</h3>
      <div className="card-content">{children}</div>
    </div>
  );
}
```

`ReactNode` is the type for anything that can be rendered: strings, numbers, elements, arrays, fragments, etc.

**Pattern 3: Callback props**

```tsx
interface TodoItemProps {
  todo: Todo;
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}

function TodoItem({ todo, onToggle, onDelete }: TodoItemProps) {
  return (
    <div>
      <input 
        type="checkbox" 
        checked={todo.completed}
        onChange={() => onToggle(todo.id)}
      />
      <span>{todo.text}</span>
      <button onClick={() => onDelete(todo.id)}>Delete</button>
    </div>
  );
}
```

**Pattern 4: Extending HTML element props**

Sometimes you want your component to accept all standard HTML attributes:

```tsx
import { ButtonHTMLAttributes } from 'react';

interface CustomButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary';
}

function CustomButton({ variant = 'primary', children, ...props }: CustomButtonProps) {
  return (
    <button className={`btn btn-${variant}`} {...props}>
      {children}
    </button>
  );
}

// Now you can use any standard button attribute
<CustomButton onClick={handleClick} disabled={isLoading} type="submit">
  Submit
</CustomButton>
```

### Common Failure Modes and Their Signatures

#### Symptom: "Object is possibly 'null'" or "Object is possibly 'undefined'"

**Console pattern**:

```bash
error TS2531: Object is possibly 'null'.
```

**Root cause**: You're accessing a property on a value that might be null or undefined.

**Solution**: Add a null check before accessing the property:

```tsx
// ❌ Error
<h1>{user.name}</h1>

// ✅ Fixed
if (!user) return null;
<h1>{user.name}</h1>

// ✅ Or use optional chaining
<h1>{user?.name}</h1>
```

#### Symptom: "Type 'X' is not assignable to type 'Y'"

**Console pattern**:

```bash
error TS2322: Type 'string' is not assignable to type 'number'.
```

**Root cause**: You're passing a value of the wrong type.

**Solution**: Convert the value or fix the type annotation:

```tsx
// ❌ Error
<Counter count="5" />

// ✅ Fixed - convert to number
<Counter count={5} />

// ✅ Or change the type to accept strings
interface CounterProps {
  count: string | number;
}
```

#### Symptom: "Property 'X' does not exist on type 'Y'"

**Console pattern**:

```bash
error TS2339: Property 'fullName' does not exist on type 'User'.
```

**Root cause**: You're accessing a property that doesn't exist in the type definition.

**Solution**: Either add the property to the type or use the correct property name:

```tsx
// ❌ Error
<h1>{user.fullName}</h1>

// ✅ Fixed - use correct property
<h1>{user.name}</h1>

// ✅ Or add to type definition
interface User {
  name: string;
  fullName: string; // Add this
}
```

### When to Apply: Type Annotation Decision Framework

**Always type**:
- Component props (always use an interface)
- State that holds complex objects (use type parameter with `useState`)
- Callback functions in props (specify parameter and return types)

**Let TypeScript infer**:
- Simple state like booleans and strings (TypeScript infers from initial value)
- Event handlers when used inline (TypeScript infers from JSX)
- Return types of components (TypeScript knows components return JSX)

**Example of good inference**:

```tsx
// ✅ Good - TypeScript infers these correctly
const [isOpen, setIsOpen] = useState(false); // inferred as boolean
const [count, setCount] = useState(0); // inferred as number
const [text, setText] = useState(''); // inferred as string

// ❌ Unnecessary - TypeScript already knows
const [isOpen, setIsOpen] = useState<boolean>(false);
```

**Example of necessary annotation**:

```tsx
// ✅ Necessary - TypeScript can't infer the future type
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<Item[]>([]);
```

### The Props Typing Journey

| Iteration | Code Pattern | TypeScript Benefit |
|-----------|-------------|-------------------|
| 0 | `function Button({ label })` | None - implicit any |
| 1 | `function Button({ label }: { label: string })` | Type safety, but verbose |
| 2 | `interface ButtonProps { label: string }` | Reusable, clear, documented |
| 3 | `interface ButtonProps extends HTMLAttributes<HTMLButtonElement>` | Full HTML support |

The progression shows increasing sophistication. Start with pattern 2 for most components. Use pattern 3 when you need full HTML element compatibility.

## Generic components

## Generic components

So far, we've typed components with specific types: `User`, `Activity`, `string`, `number`. But what if you want to build a component that works with *any* type? That's where generics come in.

### The Problem: Type-Specific Components Don't Scale

Let's say we want to build a reusable `List` component that displays items. Here's a naive approach:

```tsx
// UserList.tsx - Specific to User type
import { User } from './types';

interface UserListProps {
  items: User[];
  renderItem: (item: User) => React.ReactNode;
}

function UserList({ items, renderItem }: UserListProps) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage
<UserList 
  items={users} 
  renderItem={(user) => <span>{user.name}</span>}
/>
```

This works, but now we need separate components for every type:
- `UserList` for users
- `ActivityList` for activities
- `ProductList` for products
- `TodoList` for todos

This is repetitive. The logic is identical—only the type changes.

### The Solution: Generic Components

Generics let you write a component once and use it with any type. Here's the generic version:

```tsx
// List.tsx - Generic version
import { ReactNode } from 'react';

interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item) => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

export default List;
```

**What changed**:

1. **Generic type parameter**: `<T>` after the component name means "this component works with any type T"
2. **Type parameter in props**: `ListProps<T>` means the props interface also uses the generic type
3. **Generic array**: `items: T[]` means an array of whatever type T is
4. **Generic callback**: `renderItem: (item: T) => ReactNode` means the callback receives an item of type T

**Usage with different types**:

```tsx
import List from './List';
import { User, Activity, Product } from './types';

// With User type
<List<User>
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
  keyExtractor={(user) => user.id}
/>

// With Activity type
<List<Activity>
  items={activities}
  renderItem={(activity) => <span>{activity.action}</span>}
  keyExtractor={(activity) => activity.id}
/>

// With Product type
<List<Product>
  items={products}
  renderItem={(product) => <span>{product.title}</span>}
  keyExtractor={(product) => product.id}
/>
```

**The magic**: TypeScript infers the type from the `items` prop. When you pass `users: User[]`, TypeScript knows `T` is `User`. The `renderItem` callback then knows it receives a `User`, so you get autocomplete for `user.name`, `user.email`, etc.

### Iteration 1: Building a Generic Table Component

Let's build something more practical: a generic table component. This is a common pattern in real applications.

**The naive approach** (without generics):

```tsx
// UserTable.tsx - Specific to User type
interface Column {
  header: string;
  accessor: string; // ❌ Problem: string doesn't give us type safety
}

interface UserTableProps {
  data: User[];
  columns: Column[];
}

function UserTable({ data, columns }: UserTableProps) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => (
            <th key={col.accessor}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map((row, i) => (
          <tr key={i}>
            {columns.map((col) => (
              <td key={col.accessor}>
                {row[col.accessor]} {/* ❌ TypeScript error: Element implicitly has an 'any' type */}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**The failure**: TypeScript can't verify that `col.accessor` is a valid property of `User`. We lose type safety.

**The generic approach**:

```tsx
// Table.tsx - Generic version
import { ReactNode } from 'react';

interface Column<T> {
  header: string;
  accessor: keyof T; // ✅ Must be a key of T
  render?: (value: T[keyof T], row: T) => ReactNode;
}

interface TableProps<T> {
  data: T[];
  columns: Column<T>[];
  keyExtractor: (item: T) => string;
}

function Table<T>({ data, columns, keyExtractor }: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => (
            <th key={String(col.accessor)}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map((row) => (
          <tr key={keyExtractor(row)}>
            {columns.map((col) => (
              <td key={String(col.accessor)}>
                {col.render 
                  ? col.render(row[col.accessor], row)
                  : String(row[col.accessor])
                }
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

export default Table;
```

**Key concepts**:

1. **`keyof T`**: This means "any key that exists on type T"
   - If `T` is `User`, then `keyof T` is `"id" | "name" | "email" | "avatar" | "joinDate"`
   - TypeScript prevents you from using invalid keys

2. **`T[keyof T]`**: This means "the type of any property value on T"
   - If you access `user["name"]`, the type is `string`
   - If you access `user["id"]`, the type is `string`

3. **Optional render function**: `render?: (value: T[keyof T], row: T) => ReactNode`
   - Allows custom rendering for complex cells
   - Receives both the cell value and the entire row

**Usage**:

```tsx
import Table from './Table';
import { User } from './types';

const userColumns: Column<User>[] = [
  { header: 'Name', accessor: 'name' },
  { header: 'Email', accessor: 'email' },
  { 
    header: 'Avatar', 
    accessor: 'avatar',
    render: (value) => <img src={value as string} alt="Avatar" width={32} />
  },
  { header: 'Joined', accessor: 'joinDate' }
];

<Table<User>
  data={users}
  columns={userColumns}
  keyExtractor={(user) => user.id}
/>
```

**What TypeScript prevents**:

```tsx
// ❌ Error: 'fullName' is not a key of User
const columns: Column<User>[] = [
  { header: 'Name', accessor: 'fullName' }
];
```

**Terminal Output**:

```bash
error TS2322: Type '"fullName"' is not assignable to type 'keyof User'.
```

TypeScript tells you exactly which keys are valid: `"id" | "name" | "email" | "avatar" | "joinDate"`.

### Iteration 2: Generic Form Field Component

Another common use case: form fields that work with any data type. Let's build a generic `Select` component:

```tsx
// Select.tsx - Generic select component
import { ChangeEvent } from 'react';

interface SelectProps<T> {
  options: T[];
  value: T;
  onChange: (value: T) => void;
  getOptionLabel: (option: T) => string;
  getOptionValue: (option: T) => string;
  placeholder?: string;
}

function Select<T>({
  options,
  value,
  onChange,
  getOptionLabel,
  getOptionValue,
  placeholder
}: SelectProps<T>) {
  const handleChange = (e: ChangeEvent<HTMLSelectElement>) => {
    const selectedValue = e.target.value;
    const selectedOption = options.find(
      (option) => getOptionValue(option) === selectedValue
    );
    if (selectedOption) {
      onChange(selectedOption);
    }
  };

  return (
    <select value={getOptionValue(value)} onChange={handleChange}>
      {placeholder && <option value="">{placeholder}</option>}
      {options.map((option) => (
        <option key={getOptionValue(option)} value={getOptionValue(option)}>
          {getOptionLabel(option)}
        </option>
      ))}
    </select>
  );
}

export default Select;
```

**Usage with different types**:

```tsx
import Select from './Select';

// With User objects
interface User {
  id: string;
  name: string;
  email: string;
}

const [selectedUser, setSelectedUser] = useState<User>(users[0]);

<Select<User>
  options={users}
  value={selectedUser}
  onChange={setSelectedUser}
  getOptionLabel={(user) => user.name}
  getOptionValue={(user) => user.id}
  placeholder="Select a user"
/>

// With simple string array
const [selectedColor, setSelectedColor] = useState<string>('red');

<Select<string>
  options={['red', 'green', 'blue']}
  value={selectedColor}
  onChange={setSelectedColor}
  getOptionLabel={(color) => color}
  getOptionValue={(color) => color}
/>

// With complex Product objects
interface Product {
  id: string;
  title: string;
  price: number;
}

const [selectedProduct, setSelectedProduct] = useState<Product>(products[0]);

<Select<Product>
  options={products}
  value={selectedProduct}
  onChange={setSelectedProduct}
  getOptionLabel={(product) => `${product.title} - $${product.price}`}
  getOptionValue={(product) => product.id}
/>
```

**The power of generics**: One component works with any data type. TypeScript ensures:
- `getOptionLabel` receives the correct type
- `getOptionValue` receives the correct type
- `onChange` receives the correct type
- You get autocomplete for all properties

### Iteration 3: Generic Data Fetching Hook

Generics aren't just for components—they're powerful in custom hooks too. Let's build a generic data fetching hook:

```tsx
// useApi.ts - Generic data fetching hook
import { useState, useEffect } from 'react';

interface UseApiResult<T> {
  data: T | null;
  isLoading: boolean;
  error: string | null;
  refetch: () => void;
}

function useApi<T>(url: string): UseApiResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [isLoading, setIsLoading] = useState<boolean>(true);
  const [error, setError] = useState<string | null>(null);
  const [refetchTrigger, setRefetchTrigger] = useState<number>(0);

  useEffect(() => {
    async function fetchData() {
      setIsLoading(true);
      setError(null);
      
      try {
        const response = await fetch(url);
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        const result: T = await response.json();
        setData(result);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error');
      } finally {
        setIsLoading(false);
      }
    }

    fetchData();
  }, [url, refetchTrigger]);

  const refetch = () => {
    setRefetchTrigger((prev) => prev + 1);
  };

  return { data, isLoading, error, refetch };
}

export default useApi;
```

**Usage**:

```tsx
import useApi from './useApi';
import { User, Activity } from './types';

function UserDashboard({ userId }: { userId: string }) {
  // TypeScript knows data is User | null
  const { data: user, isLoading: userLoading, error: userError } = 
    useApi<User>(`/api/users/${userId}`);
  
  // TypeScript knows data is Activity[] | null
  const { data: activities, isLoading: activitiesLoading, error: activitiesError } = 
    useApi<Activity[]>(`/api/users/${userId}/activities`);

  if (userLoading || activitiesLoading) {
    return <LoadingSpinner />;
  }

  if (userError || activitiesError) {
    return <ErrorMessage message={userError || activitiesError || ''} />;
  }

  if (!user || !activities) {
    return null;
  }

  // TypeScript knows user is User and activities is Activity[]
  return (
    <div>
      <h1>{user.name}</h1>
      <ul>
        {activities.map((activity) => (
          <li key={activity.id}>{activity.action}</li>
        ))}
      </ul>
    </div>
  );
}
```

**The benefit**: One hook works for any API endpoint. TypeScript infers the correct return type based on the generic parameter.

### Generic Constraints: When T Needs Requirements

Sometimes you need to constrain what types can be used with a generic. For example, our `Table` component assumes every item has an `id` property for the key. Let's enforce that:

```tsx
// Table.tsx - With generic constraint
interface HasId {
  id: string;
}

interface Column<T extends HasId> {
  header: string;
  accessor: keyof T;
  render?: (value: T[keyof T], row: T) => ReactNode;
}

interface TableProps<T extends HasId> {
  data: T[];
  columns: Column<T>[];
}

function Table<T extends HasId>({ data, columns }: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => (
            <th key={String(col.accessor)}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map((row) => (
          <tr key={row.id}> {/* ✅ TypeScript knows row has id */}
            {columns.map((col) => (
              <td key={String(col.accessor)}>
                {col.render 
                  ? col.render(row[col.accessor], row)
                  : String(row[col.accessor])
                }
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**What changed**:

1. **Constraint**: `T extends HasId` means "T must be a type that has an `id` property"
2. **Guaranteed property**: We can now safely use `row.id` without a custom `keyExtractor`

**What TypeScript prevents**:

```tsx
interface Product {
  sku: string; // No 'id' property
  title: string;
}

// ❌ Error: Product doesn't satisfy the constraint
<Table<Product> data={products} columns={productColumns} />
```

**Terminal Output**:

```bash
error TS2344: Type 'Product' does not satisfy the constraint 'HasId'.
  Property 'id' is missing in type 'Product' but required in type 'HasId'.
```

### Common Generic Patterns

**Pattern 1: Generic with default type**

```tsx
interface ListProps<T = string> {
  items: T[];
  renderItem: (item: T) => ReactNode;
}

// Can be used without specifying type (defaults to string)
<List items={['a', 'b', 'c']} renderItem={(item) => <span>{item}</span>} />

// Or with explicit type
<List<User> items={users} renderItem={(user) => <span>{user.name}</span>} />
```

**Pattern 2: Multiple generic parameters**

```tsx
interface KeyValuePair<K, V> {
  key: K;
  value: V;
}

interface MapProps<K, V> {
  items: KeyValuePair<K, V>[];
  renderItem: (key: K, value: V) => ReactNode;
}

function Map<K, V>({ items, renderItem }: MapProps<K, V>) {
  return (
    <div>
      {items.map((item) => (
        <div key={String(item.key)}>
          {renderItem(item.key, item.value)}
        </div>
      ))}
    </div>
  );
}
```

**Pattern 3: Generic with union constraint**

```tsx
type Status = 'idle' | 'loading' | 'success' | 'error';

interface AsyncState<T, E = Error> {
  status: Status;
  data: T | null;
  error: E | null;
}

function useAsyncState<T, E = Error>(
  initialData: T | null = null
): [AsyncState<T, E>, (data: T) => void, (error: E) => void] {
  const [state, setState] = useState<AsyncState<T, E>>({
    status: 'idle',
    data: initialData,
    error: null
  });

  const setData = (data: T) => {
    setState({ status: 'success', data, error: null });
  };

  const setError = (error: E) => {
    setState({ status: 'error', data: null, error });
  };

  return [state, setData, setError];
}
```

### When to Use Generics: Decision Framework

**Use generics when**:
- You're building a reusable component that works with multiple types
- The logic is identical regardless of the type
- You want type safety for the specific type being used
- Examples: List, Table, Select, Modal, data fetching hooks

**Don't use generics when**:
- The component is specific to one type (just use that type directly)
- The logic differs significantly between types (use separate components)
- It makes the code harder to understand (generics add complexity)
- You're just starting to learn TypeScript (master basic types first)

**Code characteristics**:

| Approach | Setup Complexity | Type Safety | Reusability | Maintenance |
|----------|-----------------|-------------|-------------|-------------|
| Type-specific | Low | High | Low | Easy |
| Generic | Medium | High | High | Medium |
| `any` type | Low | None | High | Hard |

**When to choose each**:
- **Type-specific**: Single-use components, domain-specific logic
- **Generic**: Reusable UI components, utility functions, custom hooks
- **`any` type**: Never (there's always a better option)

### The Generic Components Journey

| Iteration | Pattern | Benefit | Cost |
|-----------|---------|---------|------|
| 0 | `UserList`, `ActivityList`, `ProductList` | Simple, clear | Repetitive code |
| 1 | `List<T>` with generic type | Reusable, type-safe | Slightly more complex |
| 2 | `List<T extends HasId>` with constraint | Guaranteed properties | More restrictive |
| 3 | `List<T, K>` with multiple generics | Maximum flexibility | Highest complexity |

Start with iteration 1 for most cases. Add constraints (iteration 2) when you need guaranteed properties. Use multiple generics (iteration 3) only when truly necessary.

## The 20% of TypeScript that gives you 80% of the value

## The 20% of TypeScript that gives you 80% of the value

TypeScript has hundreds of features. You don't need to know them all. This section covers the essential patterns that solve 80% of real-world React problems.

### The Core Toolkit: 8 Patterns You'll Use Daily

Let's consolidate what we've learned into a practical reference. These are the patterns you'll use in almost every React component.

### Pattern 1: Interface for Component Props

**The pattern**:

```tsx
interface ComponentNameProps {
  requiredProp: string;
  optionalProp?: number;
  callback: (value: string) => void;
  children?: ReactNode;
}

function ComponentName({ 
  requiredProp, 
  optionalProp = 10, 
  callback,
  children 
}: ComponentNameProps) {
  // ...
}
```

**When to use**: Every component with props (which is most components).

**Why it matters**: Clear contract, autocomplete, prevents prop mistakes.

### Pattern 2: Union Types for Variants

**The pattern**:

```tsx
type ButtonVariant = 'primary' | 'secondary' | 'danger';
type Size = 'small' | 'medium' | 'large';

interface ButtonProps {
  variant: ButtonVariant;
  size?: Size;
  children: ReactNode;
}

function Button({ variant, size = 'medium', children }: ButtonProps) {
  return (
    <button className={`btn btn-${variant} btn-${size}`}>
      {children}
    </button>
  );
}
```

**When to use**: When a prop has a fixed set of valid values.

**Why it matters**: TypeScript prevents typos like `variant="primry"`.

### Pattern 3: Nullable State with Type Parameter

**The pattern**:

```tsx
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<Item[]>([]);
const [error, setError] = useState<string | null>(null);
```

**When to use**: State that starts empty and gets populated later.

**Why it matters**: TypeScript forces you to handle the null case.

### Pattern 4: Type Guards for Null Checks

**The pattern**:

```tsx
function UserProfile({ user }: { user: User | null }) {
  // Type guard - narrows type from User | null to User
  if (!user) {
    return <div>No user</div>;
  }
  
  // TypeScript knows user is not null here
  return <h1>{user.name}</h1>;
}
```

**When to use**: Whenever you have nullable values.

**Why it matters**: Prevents "Cannot read property of null" errors.

### Pattern 5: Event Handler Types

**The pattern**:

```tsx
import { ChangeEvent, FormEvent, MouseEvent } from 'react';

function Form() {
  const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
    console.log(e.target.value);
  };
  
  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
  };
  
  const handleClick = (e: MouseEvent<HTMLButtonElement>) => {
    console.log('Clicked');
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleChange} />
      <button onClick={handleClick}>Submit</button>
    </form>
  );
}
```

**When to use**: Event handlers that need to access event properties.

**Why it matters**: Correct types for `e.target`, `e.preventDefault()`, etc.

**Pro tip**: For inline handlers, let TypeScript infer:

```tsx
<input onChange={(e) => console.log(e.target.value)} />
```

### Pattern 6: Extending HTML Element Props

**The pattern**:

```tsx
import { ButtonHTMLAttributes } from 'react';

interface CustomButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary';
  isLoading?: boolean;
}

function CustomButton({ 
  variant = 'primary', 
  isLoading = false,
  children,
  disabled,
  ...props 
}: CustomButtonProps) {
  return (
    <button 
      className={`btn btn-${variant}`}
      disabled={disabled || isLoading}
      {...props}
    >
      {isLoading ? 'Loading...' : children}
    </button>
  );
}

// Now supports all standard button props
<CustomButton onClick={handleClick} type="submit" aria-label="Submit form">
  Submit
</CustomButton>
```

**When to use**: Wrapping HTML elements with custom styling/behavior.

**Why it matters**: Supports all standard HTML attributes without listing them manually.

**Common element types**:
- `ButtonHTMLAttributes<HTMLButtonElement>`
- `InputHTMLAttributes<HTMLInputElement>`
- `HTMLAttributes<HTMLDivElement>`
- `AnchorHTMLAttributes<HTMLAnchorElement>`

### Pattern 7: Generic Components

**The pattern**:

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item) => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage
<List<User>
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
  keyExtractor={(user) => user.id}
/>
```

**When to use**: Reusable components that work with any data type.

**Why it matters**: Write once, use with any type, maintain type safety.

### Pattern 8: Custom Hook Return Types

**The pattern**:

```tsx
interface UseToggleReturn {
  isOn: boolean;
  toggle: () => void;
  setOn: () => void;
  setOff: () => void;
}

function useToggle(initialValue = false): UseToggleReturn {
  const [isOn, setIsOn] = useState(initialValue);
  
  const toggle = () => setIsOn((prev) => !prev);
  const setOn = () => setIsOn(true);
  const setOff = () => setIsOn(false);
  
  return { isOn, toggle, setOn, setOff };
}

// Usage
const { isOn, toggle } = useToggle();
```

**When to use**: Custom hooks that return multiple values.

**Why it matters**: Clear API, autocomplete for return values.

**Alternative pattern** (tuple return):

```tsx
function useToggle(initialValue = false): [boolean, () => void] {
  const [isOn, setIsOn] = useState(initialValue);
  const toggle = () => setIsOn((prev) => !prev);
  return [isOn, toggle];
}

// Usage (like useState)
const [isOn, toggle] = useToggle();
```

### Advanced Patterns You'll Need Occasionally

These patterns solve specific problems. Learn them when you encounter the problem.

### Pattern 9: Discriminated Unions for State Machines

**The problem**: State with multiple related values that should change together.

**The pattern**:

```tsx
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

function UserProfile() {
  const [state, setState] = useState<AsyncState<User>>({ status: 'idle' });
  
  // TypeScript knows which properties exist based on status
  if (state.status === 'loading') {
    return <LoadingSpinner />;
  }
  
  if (state.status === 'error') {
    return <ErrorMessage message={state.error} />; // ✅ error exists here
  }
  
  if (state.status === 'success') {
    return <h1>{state.data.name}</h1>; // ✅ data exists here
  }
  
  return <button onClick={fetchUser}>Load User</button>;
}
```

**Why it matters**: Impossible states become impossible. You can't have `status: 'success'` without `data`.

### Pattern 10: Type Assertions (Use Sparingly)

**The problem**: You know more about a type than TypeScript does.

**The pattern**:

```tsx
// When you know the API always returns this shape
const response = await fetch('/api/user');
const user = await response.json() as User;

// When you know an element exists
const input = document.getElementById('email') as HTMLInputElement;
console.log(input.value);

// When narrowing from unknown
function handleError(error: unknown) {
  if (error instanceof Error) {
    console.log(error.message);
  } else {
    console.log(String(error));
  }
}
```

**When to use**: Rarely. Only when you're certain about the type.

**Why it's dangerous**: You're telling TypeScript "trust me" and disabling type checking.

**Better alternative**: Use type guards instead:

```tsx
// ❌ Type assertion (unsafe)
const user = data as User;

// ✅ Type guard (safe)
function isUser(data: unknown): data is User {
  return (
    typeof data === 'object' &&
    data !== null &&
    'id' in data &&
    'name' in data &&
    'email' in data
  );
}

if (isUser(data)) {
  console.log(data.name); // TypeScript knows data is User
}
```

### Pattern 11: Utility Types

TypeScript provides built-in utility types for common transformations:

**`Partial<T>` - Make all properties optional**:

```tsx
interface User {
  id: string;
  name: string;
  email: string;
}

// For update functions where not all fields are required
function updateUser(id: string, updates: Partial<User>) {
  // updates can have any subset of User properties
}

updateUser('123', { name: 'Alice' }); // ✅ Valid
updateUser('123', { email: 'alice@example.com' }); // ✅ Valid
```

**`Pick<T, K>` - Select specific properties**:

```tsx
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
}

// Only expose safe properties
type PublicUser = Pick<User, 'id' | 'name' | 'email'>;

function displayUser(user: PublicUser) {
  // user.password doesn't exist here
  return <div>{user.name}</div>;
}
```

**`Omit<T, K>` - Exclude specific properties**:

```tsx
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
}

// Exclude sensitive properties
type SafeUser = Omit<User, 'password'>;

function UserCard({ user }: { user: SafeUser }) {
  return <div>{user.name}</div>;
}
```

**`Record<K, T>` - Object with specific key-value types**:

```tsx
type UserRole = 'admin' | 'editor' | 'viewer';

// Map roles to permissions
const permissions: Record<UserRole, string[]> = {
  admin: ['read', 'write', 'delete'],
  editor: ['read', 'write'],
  viewer: ['read']
};
```

### Pattern 12: Const Assertions for Literal Types

**The problem**: TypeScript widens types to be more general than you want.

**The pattern**:

```tsx
// ❌ Without const assertion
const colors = ['red', 'green', 'blue'];
// Type: string[]

// ✅ With const assertion
const colors = ['red', 'green', 'blue'] as const;
// Type: readonly ["red", "green", "blue"]

type Color = typeof colors[number];
// Type: "red" | "green" | "blue"

// Usage
interface ThemeProps {
  color: Color;
}

function Theme({ color }: ThemeProps) {
  return <div style={{ color }}>{color}</div>;
}

<Theme color="red" /> // ✅ Valid
<Theme color="purple" /> // ❌ Error
```

**When to use**: When you want exact literal types from arrays or objects.

### What You Can Safely Ignore (For Now)

TypeScript has many advanced features. You don't need these for React development:

**Ignore these until you have a specific need**:
- Conditional types (`T extends U ? X : Y`)
- Mapped types (`{ [K in keyof T]: T[K] }`)
- Template literal types (`` `${string}-${number}` ``)
- Decorators (experimental)
- Namespaces (legacy)
- Enums (use union types instead)
- Abstract classes (use interfaces)

**Why ignore them**: They solve edge cases. Learn the core patterns first.

### The Essential TypeScript Checklist

Before considering yourself proficient with TypeScript in React, ensure you can:

- ✅ Type component props with interfaces
- ✅ Use union types for variants (`'small' | 'medium' | 'large'`)
- ✅ Type state with `useState<T>`
- ✅ Handle nullable values with type guards
- ✅ Type event handlers (`ChangeEvent`, `FormEvent`, etc.)
- ✅ Extend HTML element props when needed
- ✅ Build generic components for reusable logic
- ✅ Type custom hook return values
- ✅ Use utility types (`Partial`, `Pick`, `Omit`)
- ✅ Read and understand TypeScript error messages

If you can do these 10 things, you're ready for production React development with TypeScript.

### The TypeScript Learning Path

**Week 1: Basics**
- Type component props
- Type state with `useState`
- Handle nullable values

**Week 2: Events and Forms**
- Type event handlers
- Build typed forms
- Use union types for variants

**Week 3: Reusability**
- Build generic components
- Type custom hooks
- Extend HTML element props

**Week 4: Advanced Patterns**
- Discriminated unions
- Utility types
- Type guards

**Month 2+: Practice**
- Convert existing components to TypeScript
- Build a typed component library
- Contribute to typed open-source projects

### Common Mistakes and How to Avoid Them

**Mistake 1: Using `any` to silence errors**

```tsx
// ❌ Bad - defeats the purpose of TypeScript
const [data, setData] = useState<any>(null);

// ✅ Good - be explicit about the type
const [data, setData] = useState<User | null>(null);
```

**Mistake 2: Not handling null/undefined**

```tsx
// ❌ Bad - will crash if user is null
<h1>{user.name}</h1>

// ✅ Good - handle the null case
if (!user) return null;
<h1>{user.name}</h1>

// ✅ Or use optional chaining
<h1>{user?.name ?? 'Unknown'}</h1>
```

**Mistake 3: Over-typing simple things**

```tsx
// ❌ Unnecessary - TypeScript infers this
const [count, setCount] = useState<number>(0);

// ✅ Better - let TypeScript infer
const [count, setCount] = useState(0);
```

**Mistake 4: Not using union types for variants**

```tsx
// ❌ Bad - any string is valid
interface ButtonProps {
  variant: string;
}

// ✅ Good - only specific strings are valid
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'danger';
}
```

**Mistake 5: Ignoring TypeScript errors**

```tsx
// ❌ Bad - suppressing errors
// @ts-ignore
const value = user.name;

// ✅ Good - fix the underlying issue
const value = user?.name ?? 'Unknown';
```

### The 80/20 Summary

**The 20% you need to know**:
1. Interface for props
2. Union types for variants
3. `useState<T>` for complex state
4. Type guards for null checks
5. Event handler types
6. Extending HTML props
7. Generic components
8. Custom hook return types

**The 80% of problems this solves**:
- Prop mistakes
- Null/undefined errors
- Event handler errors
- API response mismatches
- Refactoring safety
- Autocomplete and IntelliSense
- Documentation through types
- Onboarding new developers

**Time investment**:
- Learning: 1-2 weeks
- Proficiency: 1-2 months
- Mastery: 6-12 months

**Return on investment**:
- 38% fewer bugs (Airbnb study)
- 15% faster development after learning curve (Microsoft)
- Significantly easier refactoring
- Better developer experience

### Final Implementation: The Complete Typed Dashboard

Let's see our User Dashboard with all TypeScript patterns applied:

```typescript
// types.ts - Central type definitions
export interface User {
  id: string;
  name: string;
  email: string;
  avatar: string;
  joinDate: string;
}

export interface Activity {
  id: string;
  timestamp: string;
  action: string;
  target: string;
}

export type LoadingState = 'idle' | 'loading' | 'success' | 'error';

export interface AsyncData<T> {
  data: T | null;
  status: LoadingState;
  error: string | null;
}
```

```tsx
// UserDashboard.tsx - Main component
import { useState, useEffect } from 'react';
import { User, Activity, AsyncData } from './types';
import UserProfile from './UserProfile';
import ActivityFeed from './ActivityFeed';
import LoadingSpinner from './LoadingSpinner';
import ErrorMessage from './ErrorMessage';

interface UserDashboardProps {
  userId: string;
}

function UserDashboard({ userId }: UserDashboardProps) {
  const [userData, setUserData] = useState<AsyncData<User>>({
    data: null,
    status: 'idle',
    error: null
  });

  const [activitiesData, setActivitiesData] = useState<AsyncData<Activity[]>>({
    data: null,
    status: 'idle',
    error: null
  });

  useEffect(() => {
    async function fetchData() {
      setUserData((prev) => ({ ...prev, status: 'loading' }));
      setActivitiesData((prev) => ({ ...prev, status: 'loading' }));

      try {
        const [userRes, activitiesRes] = await Promise.all([
          fetch(`/api/users/${userId}`),
          fetch(`/api/users/${userId}/activities`)
        ]);

        if (!userRes.ok || !activitiesRes.ok) {
          throw new Error('Failed to fetch data');
        }

        const user: User = await userRes.json();
        const activities: Activity[] = await activitiesRes.json();

        setUserData({ data: user, status: 'success', error: null });
        setActivitiesData({ data: activities, status: 'success', error: null });
      } catch (err) {
        const errorMessage = err instanceof Error ? err.message : 'Unknown error';
        setUserData({ data: null, status: 'error', error: errorMessage });
        setActivitiesData({ data: null, status: 'error', error: errorMessage });
      }
    }

    fetchData();
  }, [userId]);

  if (userData.status === 'loading' || activitiesData.status === 'loading') {
    return <LoadingSpinner size="large" />;
  }

  if (userData.status === 'error' || activitiesData.status === 'error') {
    return <ErrorMessage message={userData.error || activitiesData.error || 'Error'} />;
  }

  if (!userData.data || !activitiesData.data) {
    return <div>No data available</div>;
  }

  return (
    <div className="dashboard">
      <UserProfile user={userData.data} />
      <ActivityFeed activities={activitiesData.data} />
    </div>
  );
}

export default UserDashboard;
```

**What this demonstrates**:
- ✅ Typed props with interface
- ✅ Complex state with custom types
- ✅ Discriminated union for async state
- ✅ Type guards for null checks
- ✅ Proper error handling with type narrowing
- ✅ Type-safe API responses
- ✅ Clear component contracts

This is production-ready TypeScript React code. It's type-safe, maintainable, and catches errors at compile time instead of runtime.

### Lessons Learned: From JavaScript to TypeScript

**The journey**:
1. **JavaScript**: Fast to write, risky to maintain
2. **TypeScript (basic)**: Slower to write, safer to maintain
3. **TypeScript (proficient)**: Fast to write, safe to maintain

**The mindset shift**:
- JavaScript: "Does this code run?"
- TypeScript: "Does this code make sense?"

**The payoff**:
- Fewer runtime errors
- Better developer experience
- Easier refactoring
- Self-documenting code
- Faster onboarding

**The cost**:
- Initial learning curve (1-2 weeks)
- Slightly more verbose code
- Occasional fights with the type system

**The verdict**: For any React project that will be maintained for more than a few months, TypeScript is worth it. The initial friction pays dividends in long-term maintainability.

You now have the essential TypeScript knowledge for professional React development. The next chapters will build on this foundation, using TypeScript throughout.
