# Chapter 20: Styling in Next.js

## Tailwind CSS: the pragmatic choice

## The Styling Challenge

In modern web development, styling is more than just making things look good. It's about creating maintainable, scalable, and performant design systems. A poor styling strategy can lead to bloated CSS files, conflicting class names, and a frustrating developer experience.

In this chapter, we'll explore the modern, pragmatic approach to styling in Next.js. We'll start with the industry standard, Tailwind CSS, and build upon it with other powerful tools.

### Phase 1: Establish the Reference Implementation

To ground our exploration, we'll build a single, realistic component: a `UserProfileCard`. We will evolve this component's styling throughout the chapter, demonstrating the pros and cons of each approach.

Our starting point is a completely unstyled, functional React component. This is our "problem state."

**Project Structure**:
```
src/
└── app/
    ├── components/
    │   └── UserProfileCard.tsx  <-- Our anchor component
    └── page.tsx
```

Here is the initial, unstyled code.

```tsx
// src/components/UserProfileCard.tsx

type UserProfileCardProps = {
  user: {
    name: string;
    username: string;

    avatarUrl: string;
    bio: string;
    stats: {
      followers: number;
      following: number;
    };
  };
};

export function UserProfileCard({ user }: UserProfileCardProps) {
  return (
    <div>
      <img src={user.avatarUrl} alt={`${user.name}'s avatar`} />
      <div>
        <h2>{user.name}</h2>
        <p>@{user.username}</p>
      </div>
      <p>{user.bio}</p>
      <div>
        <div>
          <span>{user.stats.followers}</span>
          <span>Followers</span>
        </div>
        <div>
          <span>{user.stats.following}</span>
          <span>Following</span>
        </div>
      </div>
      <button>Follow</button>
    </div>
  );
}
```

```tsx
// src/app/page.tsx

import { UserProfileCard } from './components/UserProfileCard';

const sampleUser = {
  name: 'Ada Lovelace',
  username: 'ada',
  avatarUrl: 'https://via.placeholder.com/150',
  bio: 'The first computer programmer and an English mathematician and writer.',
  stats: {
    followers: 1815,
    following: 10,
  },
};

export default function HomePage() {
  return (
    <main>
      <UserProfileCard user={sampleUser} />
    </main>
  );
}
```

### Diagnostic Analysis: Reading the Failure

Running this code gives us a functional but visually unappealing result.

**Browser Behavior**:
The browser displays a jumble of unstyled HTML elements. The image is too large, the text has no hierarchy, and the layout is a simple vertical stack. It's technically working, but it's not a usable UI.



**Let's parse this evidence**:

1.  **What the user experiences**: A raw, unformatted page that looks broken.
2.  **What the console reveals**: No errors. The code is correct, but lacks presentation.
3.  **Root cause identified**: We have written semantic HTML (and JSX), but we haven't provided any CSS rules to tell the browser how to render it.
4.  **Why the current approach can't solve this**: Plain JSX without a styling system offers no visual control.
5.  **What we need**: A systematic way to apply styles to our components. We'll start with the most popular and productive choice in the Next.js ecosystem: Tailwind CSS.

### Iteration 1: Setting Up and Applying Tailwind CSS

Tailwind CSS is a utility-first CSS framework. Instead of writing CSS files, you apply pre-existing "utility" classes directly in your JSX. This approach promotes co-location of styles and logic, speeds up development, and enforces a consistent design system.

#### Step 1: Installation and Setup

First, we'll install Tailwind CSS and its dependencies, then initialize it for our Next.js project.

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

This creates two configuration files: `tailwind.config.ts` and `postcss.config.js`.

#### Step 2: Configure Template Paths

Next, we need to tell Tailwind where our component and page files are so it can scan them for class names and generate the necessary CSS.

```typescript
// tailwind.config.ts

import type { Config } from 'tailwindcss'

const config: Config = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}', // <-- Make sure this is included
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
export default config
```

#### Step 3: Add Tailwind Directives to Global CSS

Finally, we add Tailwind's base styles, component classes, and utility classes to our global CSS file. Next.js creates this for you at `src/app/globals.css`.

```css
/* src/app/globals.css */

@tailwind base;
@tailwind components;
@tailwind utilities;

/* You can add any other global styles below */
```

With the setup complete, we can now refactor our `UserProfileCard` to use Tailwind's utility classes.

#### Step 4: Applying Utility Classes

We'll go through our component and add classes to style each element.

**Before** (Unstyled):

```tsx
// src/components/UserProfileCard.tsx (Initial Version)

export function UserProfileCard({ user }: UserProfileCardProps) {
  return (
    <div>
      <img src={user.avatarUrl} alt={`${user.name}'s avatar`} />
      <div>
        <h2>{user.name}</h2>
        <p>@{user.username}</p>
      </div>
      <p>{user.bio}</p>
      <div>
        <div>
          <span>{user.stats.followers}</span>
          <span>Followers</span>
        </div>
        <div>
          <span>{user.stats.following}</span>
          <span>Following</span>
        </div>
      </div>
      <button>Follow</button>
    </div>
  );
}
```

**After** (Styled with Tailwind):

```tsx
// src/components/UserProfileCard.tsx (Tailwind Version)

type UserProfileCardProps = {
  user: {
    name: string;
    username: string;
    avatarUrl: string;
    bio: string;
    stats: {
      followers: number;
      following: number;
    };
  };
};

export function UserProfileCard({ user }: UserProfileCardProps) {
  return (
    <div className="max-w-sm mx-auto bg-white rounded-xl shadow-md overflow-hidden md:max-w-2xl">
      <div className="md:flex">
        <div className="md:shrink-0">
          <img 
            className="h-48 w-full object-cover md:h-full md:w-48" 
            src={user.avatarUrl} 
            alt={`${user.name}'s avatar`} 
          />
        </div>
        <div className="p-8">
          <div className="uppercase tracking-wide text-sm text-indigo-500 font-semibold">
            @{user.username}
          </div>
          <h2 className="block mt-1 text-lg leading-tight font-medium text-black">
            {user.name}
          </h2>
          <p className="mt-2 text-slate-500">{user.bio}</p>
          <div className="mt-4 flex space-x-4">
            <div>
              <span className="font-bold text-black">{user.stats.followers}</span>
              <span className="text-slate-500"> Followers</span>
            </div>
            <div>
              <span className="font-bold text-black">{user.stats.following}</span>
              <span className="text-slate-500"> Following</span>
            </div>
          </div>
          <button className="mt-6 bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded">
            Follow
          </button>
        </div>
      </div>
    </div>
  );
}
```

### Verification

**Browser Behavior**:
The component is now beautifully styled. The layout is responsive, text has proper hierarchy and color, and the button has interactive states.



**Expected vs. Actual Improvement**:
- **Expected**: The component should look like a proper UI element.
- **Actual**: The component is fully styled, responsive, and interactive, all without writing a single line of custom CSS.

### When to Apply This Solution

-   **What it optimizes for**: Development speed, design consistency, co-location of styles, and performance (Tailwind purges unused CSS for tiny production builds).
-   **What it sacrifices**: The familiarity of traditional CSS syntax for those new to utility-first frameworks.
-   **When to choose this approach**: For almost all new Next.js projects. It is the de-facto standard and integrates seamlessly.
-   **When to avoid this approach**: In projects with a pre-existing, large CSS codebase that cannot be refactored, or if your team has a strong, unmovable preference for another styling methodology.

### Limitation Preview

This works great, but look at the `button` element. Its `className` string is quite long: `mt-6 bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded`. If we use this button style elsewhere, we'd have to copy and paste this long string, which is a violation of the DRY (Don't Repeat Yourself) principle.

What if we want to create a reusable, encapsulated style for specific elements like buttons, without polluting our JSX with dozens of utility classes? This leads us to our next tool: CSS Modules.

## CSS Modules as a fallback

## Encapsulating Styles with CSS Modules

Our `UserProfileCard` is now styled with Tailwind, but we've identified a potential maintenance issue: long, repeated strings of utility classes for common elements like buttons. While Tailwind offers solutions for this (like the `@apply` directive or component plugins), Next.js has a powerful built-in feature that provides another excellent option: **CSS Modules**.

CSS Modules allow you to write standard CSS in a file, but when you import it into a component, the class names are automatically scoped to that component. This prevents class name collisions and encapsulates styles effectively.

### Iteration 2: Combining Tailwind with CSS Modules

Let's refactor our button to use CSS Modules, while leaving the rest of the component styled with Tailwind utilities. This demonstrates how the two systems can work together harmoniously.

#### Current Limitation

The button's styling is defined inline within the JSX.
```tsx
<button className="mt-6 bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded">
  Follow
</button>
```
If we need another button with the exact same style, we have to copy this entire string. If we need to update the style, we have to find and replace it everywhere.

#### New Scenario: Creating a Reusable Button Style

We want to define our "primary button" style once and apply it by referencing a single, meaningful name.

#### Failure Demonstration: The Global CSS Pitfall

A naive approach would be to create a global CSS class.

**File Structure**:
```
src/
└── app/
    ├── styles/
    │   └── buttons.css  <-- New global CSS file
    ├── components/
    │   └── UserProfileCard.tsx
    └── layout.tsx       <-- Import the CSS here
```

```css
/* src/app/styles/buttons.css */

/* 
  We are using Tailwind's @apply directive here to use Tailwind tokens,
  but the core problem is the global nature of the class name.
*/
.primary-button {
  @apply mt-6 bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded;
}
```

```tsx
// src/app/layout.tsx
import './styles/buttons.css'; // <-- Import globally

export default function RootLayout({ children }: { children: React.ReactNode }) {
  // ...
}
```

```tsx
// src/components/UserProfileCard.tsx
// ...
<button className="primary-button">
  Follow
</button>
// ...
```

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The button looks correct. The styles are applied. So what's the failure? The failure is not visual; it's architectural.

**Let's parse this evidence**:

1.  **What the user experiences**: Everything looks fine.
2.  **What the console reveals**: No errors.
3.  **Root cause identified**: We've introduced a global class name, `.primary-button`. In a small project, this is fine. In a large application with multiple developers, someone else might unknowingly define their own `.primary-button` class in another file. The last one loaded wins, leading to unpredictable styling bugs that are incredibly difficult to debug. This is known as "global namespace pollution."
4.  **Why the current approach can't solve this**: Global CSS by its very nature lacks scoping. Any class can affect any element anywhere in the application.
5.  **What we need**: A way to write CSS that is scoped *locally* to the component that imports it.

#### Technique Introduced: CSS Modules

A CSS Module is a CSS file where all class names and animation names are scoped locally by default. To use it, you name your file with the `.module.css` extension (e.g., `Button.module.css`).

#### Solution Implementation

Let's create a CSS Module specifically for our `UserProfileCard` component.

**File Structure Update**:
```
src/
└── app/
    └── components/
        ├── UserProfileCard.tsx
        └── UserProfileCard.module.css  <-- New CSS Module
```

**Before** (Global CSS class):
```tsx
// UserProfileCard.tsx
<button className="primary-button">Follow</button>
```

**After** (Using CSS Modules):

First, create the CSS Module file.

```css
/* src/app/components/UserProfileCard.module.css */

.followButton {
  /* We can use @apply to leverage Tailwind's design tokens */
  @apply mt-6 bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded;
  
  /* We can also add standard CSS properties */
  transition: transform 0.1s ease-in-out;
}

.followButton:active {
  transform: scale(0.95);
}
```

Now, import and use it in the component.

```tsx
// src/components/UserProfileCard.tsx (CSS Modules Version)
import styles from './UserProfileCard.module.css'; // <-- Import the module

type UserProfileCardProps = {
  // ... (props are the same)
};

export function UserProfileCard({ user }: UserProfileCardProps) {
  return (
    <div className="max-w-sm mx-auto bg-white rounded-xl shadow-md overflow-hidden md:max-w-2xl">
      <div className="md:flex">
        {/* ... other JSX remains the same ... */}
        <div className="p-8">
          {/* ... */}
          <button className={styles.followButton}> {/* <-- Apply the scoped class */}
            Follow
          </button>
        </div>
      </div>
    </div>
  );
}
```

### Verification

**Browser Behavior**:
The button looks identical to the previous versions, but now includes the new `transition` and `transform` effects we added.

**Browser DevTools - Elements Tab**:
If you inspect the button in the browser's developer tools, you'll see the magic of CSS Modules. The rendered class name is not `followButton`.

```html
<button class="UserProfileCard_followButton__a1B2c">Follow</button>
```

Next.js has automatically generated a unique class name (`UserProfileCard_followButton__` followed by a unique hash) to guarantee it will never conflict with any other style in your application.

**Expected vs. Actual Improvement**:
- **Expected**: To have a reusable, named style for our button.
- **Actual**: We achieved that *and* gained guaranteed style encapsulation, preventing any future global CSS conflicts.

### When to Apply This Solution

-   **What it optimizes for**: Style encapsulation, preventing global namespace pollution, and allowing the use of traditional CSS syntax for complex styles (animations, pseudo-selectors) that can be verbose with utility classes.
-   **What it sacrifices**: Co-location. The styles for the button are now in a separate file, requiring a context switch.
-   **When to choose this approach**:
    -   When you have a complex, component-specific style that would be unwieldy as a long utility string.
    -   When you need to apply styles based on dynamic logic that is easier to express by conditionally including a single class name.
    -   When integrating with third-party libraries that expect traditional CSS class names.
-   **When to avoid this approach**: As your primary styling method. Stick with Tailwind for 90% of your styling and use CSS Modules as a powerful tool for the remaining 10% where encapsulation is key.

### Limitation Preview

We've now built and styled a component from scratch using the best of both utility classes and scoped CSS. But this is still a lot of work. We had to build the card, the button, and the layout ourselves. What if we could leverage pre-built, accessible, and themeable components to build UIs even faster, without giving up control? This brings us to `shadcn/ui`.

## shadcn/ui: pre-built components done right

## Accelerating Development with shadcn/ui

So far, we've focused on writing CSS. But modern frontend development is about building with *components*. We've built our `UserProfileCard` from low-level `div`, `img`, and `button` elements. This gives us full control, but it's slow and requires us to handle details like accessibility (ARIA attributes, keyboard navigation) ourselves.

**shadcn/ui** is not a traditional component library like Material-UI or Bootstrap. Instead, it's a collection of beautifully designed, accessible, and reusable components that you copy and paste directly into your project. You own the code. This gives you the speed of a component library with the flexibility of writing it yourself.

### Iteration 3: Rebuilding with shadcn/ui

Let's see how we can dramatically simplify our `UserProfileCard` by rebuilding it with components from shadcn/ui.

#### Current Limitation

Our current implementation is bespoke. Every piece of it was hand-coded. To build another UI element, like a dialog box or a dropdown menu, we would have to start from scratch again. This doesn't scale.

#### Technique Introduced: Composable, Owned Components

The philosophy of shadcn/ui is to provide the building blocks, not a finished house. It uses Tailwind CSS for styling and Radix UI for accessibility primitives, giving you a best-in-class foundation.

#### Step 1: Initialize shadcn/ui

First, we add the shadcn/ui CLI to our project and run the `init` command. It will ask a few questions about your project setup (TypeScript, `tailwind.config.js` location, etc.).

```bash
npx shadcn-ui@latest init
```

This command does a few things:
1.  Installs necessary dependencies (`class-variance-authority`, `clsx`, `tailwind-merge`).
2.  Creates a `components/ui` directory for the components you'll add.
3.  Creates a `lib/utils.ts` file with a helper function (`cn`) for merging Tailwind classes.
4.  Updates `tailwind.config.ts` with theme variables for colors, border radius, etc.

#### Step 2: Add Required Components

Instead of installing a whole library, you add only the components you need. For our card, we need a `Card`, an `Avatar`, and a `Button`.

```bash
npx shadcn-ui@latest add card
npx shadcn-ui@latest add avatar
npx shadcn-ui@latest add button
```

Now, check your project structure. You'll see the new component files. You can open them, read them, and even modify them. You own this code.

**New Project Structure**:
```
src/
├── components/
│   ├── ui/
│   │   ├── avatar.tsx   <-- Added by CLI
│   │   ├── button.tsx   <-- Added by CLI
│   │   └── card.tsx     <-- Added by CLI
│   └── UserProfileCard.tsx
├── lib/
│   └── utils.ts         <-- Added by CLI
└── app/
    └── page.tsx
```

#### Step 3: Refactor the UserProfileCard

Now we can refactor our component to use these new, high-level building blocks.

**Before** (Manual Tailwind/CSS Modules):

```tsx
// src/components/UserProfileCard.tsx (Previous Version)
import styles from './UserProfileCard.module.css';

export function UserProfileCard({ user }: UserProfileCardProps) {
  return (
    <div className="max-w-sm mx-auto bg-white rounded-xl shadow-md overflow-hidden md:max-w-2xl">
      <div className="md:flex">
        <div className="md:shrink-0">
          <img 
            className="h-48 w-full object-cover md:h-full md:w-48" 
            src={user.avatarUrl} 
            alt={`${user.name}'s avatar`} 
          />
        </div>
        <div className="p-8">
          {/* ... a lot of divs and manual styling ... */}
          <button className={styles.followButton}>
            Follow
          </button>
        </div>
      </div>
    </div>
  );
}
```

**After** (Using shadcn/ui components):

The new version is much more declarative and readable. We're composing components, not styling divs.

```tsx
// src/components/UserProfileCard.tsx (shadcn/ui Version)

import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar";
import { Button } from "@/components/ui/button";
import { 
  Card, 
  CardContent, 
  CardDescription, 
  CardFooter, 
  CardHeader, 
  CardTitle 
} from "@/components/ui/card";

type UserProfileCardProps = {
  user: {
    name: string;
    username: string;
    avatarUrl: string;
    bio: string;
    stats: {
      followers: number;
      following: number;
    };
  };
};

export function UserProfileCard({ user }: UserProfileCardProps) {
  return (
    <Card className="max-w-sm mx-auto">
      <CardHeader className="flex flex-row items-center gap-4">
        <Avatar className="h-16 w-16">
          <AvatarImage src={user.avatarUrl} alt={`@${user.username}`} />
          <AvatarFallback>{user.name.charAt(0)}</AvatarFallback>
        </Avatar>
        <div>
          <CardTitle>{user.name}</CardTitle>
          <CardDescription>@{user.username}</CardDescription>
        </div>
      </CardHeader>
      <CardContent>
        <p>{user.bio}</p>
        <div className="mt-4 flex space-x-4">
          <div>
            <span className="font-bold">{user.stats.followers}</span>
            <span className="text-muted-foreground"> Followers</span>
          </div>
          <div>
            <span className="font-bold">{user.stats.following}</span>
            <span className="text-muted-foreground"> Following</span>
          </div>
        </div>
      </CardContent>
      <CardFooter>
        <Button className="w-full">Follow</Button>
      </CardFooter>
    </Card>
  );
}
```

### Verification

**Browser Behavior**:
The component looks similar to our manually styled version, but it's more polished and consistent with a professional design system. The `Avatar` component automatically handles fallbacks if the image fails to load. The `Button` has built-in accessibility and focus states. The `Card` components provide semantic structure.



**Expected vs. Actual Improvement**:
-   **Expected**: A faster way to build the component.
-   **Actual**: We built the component faster, with more robust and accessible code. The resulting JSX is more semantic and easier to read. We now have a system for adding other complex components to our app consistently. Notice the use of `text-muted-foreground` - this class comes from the shadcn/ui setup and uses CSS variables, which is a perfect segue into theming.

### Limitation Preview

Our application looks great... in light mode. If a user has their operating system set to dark mode, our UI doesn't adapt. It's a jarring experience. We've built a static theme. How can we make our application's appearance adapt to user preferences and allow them to toggle between themes?

## Dark mode and theming

## Dark Mode and Theming

A modern web application is incomplete without support for dark mode. Providing a theme that respects user preferences is a critical aspect of good user experience. Thanks to our choice of Tailwind CSS and shadcn/ui, implementing robust theming is surprisingly straightforward.

### Current Limitation

Our `UserProfileCard` has hardcoded colors (e.g., `bg-white`, `text-black`). These look fine in a light-themed environment but provide a poor experience in the dark. The UI doesn't respond to the user's OS-level preference for a dark color scheme.

### Technique Introduced: Class-based Theming with `next-themes`

The standard and most effective way to handle theming in Next.js is to use a class-based strategy.
1.  We'll use the `next-themes` library to manage the theme state (light, dark, or system) and apply a class (e.g., `dark`) to the `<html>` element.
2.  We'll configure Tailwind to use this class to apply different styles via the `dark:` variant.
3.  We'll use CSS variables for our colors, which shadcn/ui has already set up for us.

### Iteration 4: Implementing Dark Mode

#### Step 1: Install `next-themes`

```bash
npm install next-themes
```

#### Step 2: Configure Tailwind for Class-based Dark Mode

Open your `tailwind.config.ts` and tell Tailwind to use the `class` strategy for dark mode. The shadcn/ui `init` command should have already done this, but it's crucial to verify.

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss'

const config = {
  darkMode: ["class"], // <-- This is the key
  content: [
    // ...
  ],
  // ...
} satisfies Config

export default config
```

#### Step 3: Create a Theme Provider

Because `next-themes` uses React Context and needs to know when the component is mounted to read the user's system theme, we must use it within a Client Component. We'll create a dedicated provider component to wrap our entire application.

```tsx
// src/components/theme-provider.tsx

"use client";

import * as React from "react";
import { ThemeProvider as NextThemesProvider } from "next-themes";
import { type ThemeProviderProps } from "next-themes/dist/types";

export function ThemeProvider({ children, ...props }: ThemeProviderProps) {
  return <NextThemesProvider {...props}>{children}</NextThemesProvider>;
}
```

#### Step 4: Apply the Provider in the Root Layout

Now, wrap the `<body>` of your root layout (`src/app/layout.tsx`) with this new `ThemeProvider`. This makes the theme available to every component in your app.

```tsx
// src/app/layout.tsx

import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import { ThemeProvider } from "@/components/theme-provider"; // <-- Import

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "Create Next App",
  description: "Generated by create next app",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className={inter.className}>
        <ThemeProvider
          attribute="class"
          defaultTheme="system"
          enableSystem
          disableTransitionOnChange
        >
          {children}
        </ThemeProvider>
      </body>
    </html>
  );
}
```

#### Step 5: Create a Theme Toggle Component

To allow users to manually switch themes, we need a UI control. Let's create a simple button that toggles between light, dark, and system preferences. This component will also be a Client Component because it uses the `useTheme` hook.

```tsx
// src/components/theme-toggle.tsx

"use client";

import * as React from "react";
import { Moon, Sun } from "lucide-react";
import { useTheme } from "next-themes";

import { Button } from "@/components/ui/button";

export function ThemeToggle() {
  const { theme, setTheme } = useTheme();

  const toggleTheme = () => {
    setTheme(theme === "light" ? "dark" : "light");
  };

  return (
    <Button variant="outline" size="icon" onClick={toggleTheme}>
      <Sun className="h-[1.2rem] w-[1.2rem] rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0" />
      <Moon className="absolute h-[1.2rem] w-[1.2rem] rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100" />
      <span className="sr-only">Toggle theme</span>
    </Button>
  );
}
```

#### Step 6: Add the Toggle to the UI

Let's add our new `ThemeToggle` to the main page so we can test it.

```tsx
// src/app/page.tsx

import { UserProfileCard } from './components/UserProfileCard';
import { ThemeToggle } from './components/theme-toggle'; // <-- Import

const sampleUser = { /* ... */ };

export default function HomePage() {
  return (
    // Added a dark background for the page itself for better contrast
    <main className="min-h-screen bg-background flex flex-col items-center justify-center p-4">
      <div className="absolute top-4 right-4">
        <ThemeToggle />
      </div>
      <UserProfileCard user={sampleUser} />
    </main>
  );
}
```

### Verification

**Browser Behavior**:
The application now loads with a theme that matches your OS preference. Clicking the toggle button instantly switches between light and dark modes. All the shadcn/ui components (`Card`, `Button`, `Avatar`) automatically adapt their colors.

**Light Mode**:


**Dark Mode**:


### Banish Magic: How It Works

This feels magical, but it's a clear, mechanical process:
1.  **`ThemeProvider`**: On load, it checks `localStorage` for a saved theme or queries the OS for its preference (`prefers-color-scheme`). It then adds `class="dark"` or `class="light"` to the `<html>` tag.
2.  **`ThemeToggle`**: The `useTheme` hook communicates with the `ThemeProvider`'s context. Calling `setTheme('dark')` updates the context, which in turn changes the class on the `<html>` tag.
3.  **CSS Variables**: Your `globals.css` file (configured by shadcn/ui) defines all colors as CSS variables.
    ```css
    /* src/app/globals.css */
    :root {
      --background: 0 0% 100%; /* White */
      --foreground: 222.2 84% 4.9%; /* Black */
      /* ... other light theme colors */
    }

    .dark {
      --background: 222.2 84% 4.9%; /* Black */
      --foreground: 210 40% 98%; /* White */
      /* ... other dark theme colors */
    }
    ```
4.  **Tailwind Integration**: Tailwind utility classes like `bg-background` and `text-foreground` are configured in `tailwind.config.ts` to use these CSS variables.
    ```ts
    // tailwind.config.ts
    // ...
    theme: {
      extend: {
        colors: {
          background: "hsl(var(--background))",
          foreground: "hsl(var(--foreground))",
          // ...
        }
      }
    }
    ```
When the class on the `<html>` tag changes from `light` to `dark`, the CSS variables defined inside the `.dark` block take precedence, and every component using those variables instantly updates its appearance.

### The Journey: From Problem to Solution

```markdown
| Iteration | Problem                                     | Technique Applied      | Result                                                              | Key Insight                                                              |
| --------- | ------------------------------------------- | ---------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| 0         | Unstyled, unusable UI                       | Plain JSX/HTML         | A functional but ugly component.                                    | HTML provides structure, but CSS provides presentation.                  |
| 1         | No styling system                           | Tailwind CSS           | A fully styled, responsive component with utility classes.          | Utility-first CSS is fast and maintains co-location.                     |
| 2         | Long, un-reusable class strings             | CSS Modules            | Encapsulated, reusable style for the button, preventing conflicts.  | Scoped CSS is a powerful tool for complex, component-specific styles.    |
| 3         | Building everything from scratch is slow    | shadcn/ui              | Rebuilt with accessible, pre-made components. Drastically less code. | Don't reinvent the wheel. Use component systems you own and control.     |
| 4         | UI is stuck in light mode, ignores user prefs | `next-themes` + CSS Vars | Fully themeable UI with light/dark mode toggle.                     | A class on the root element + CSS variables is the key to modern theming. |
```

### Final Implementation

Here is our final, production-ready `UserProfileCard` component. It's built with high-level components, is fully themeable, accessible, and the code is clean and declarative.

```tsx
// src/components/UserProfileCard.tsx (Final Version)

import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar";
import { Button } from "@/components/ui/button";
import { 
  Card, 
  CardContent, 
  CardDescription, 
  CardFooter, 
  CardHeader, 
  CardTitle 
} from "@/components/ui/card";

type UserProfileCardProps = {
  user: {
    name: string;
    username: string;
    avatarUrl: string;
    bio: string;
    stats: {
      followers: number;
      following: number;
    };
  };
};

export function UserProfileCard({ user }: UserProfileCardProps) {
  return (
    <Card className="w-full max-w-sm">
      <CardHeader className="flex flex-row items-center gap-4">
        <Avatar className="h-16 w-16">
          <AvatarImage src={user.avatarUrl} alt={`@${user.username}`} />
          <AvatarFallback>{user.name.charAt(0)}</AvatarFallback>
        </Avatar>
        <div>
          <CardTitle>{user.name}</CardTitle>
          <CardDescription>@{user.username}</CardDescription>
        </div>
      </CardHeader>
      <CardContent>
        <p className="text-sm text-muted-foreground">{user.bio}</p>
        <div className="mt-4 flex divide-x rounded-lg border">
          <div className="flex-1 p-2 text-center">
            <p className="font-bold">{user.stats.followers}</p>
            <p className="text-xs text-muted-foreground">Followers</p>
          </div>
          <div className="flex-1 p-2 text-center">
            <p className="font-bold">{user.stats.following}</p>
            <p className="text-xs text-muted-foreground">Following</p>
          </div>
        </div>
      </CardContent>
      <CardFooter>
        <Button className="w-full">Follow</Button>
      </CardFooter>
    </Card>
  );
}
```

### Decision Framework: Which Styling Approach When?

| Approach              | Best For                                                                                             | Trade-offs                                                              | Recommendation                                                                                             |
| --------------------- | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **Tailwind CSS**      | The vast majority of styling needs. Layout, typography, color, spacing.                              | Can lead to long `className` strings for complex components.            | **Use as your default.** The foundation of your styling system.                                            |
| **CSS Modules**       | Complex, state-dependent styles, animations, or encapsulating a set of utilities under one name.     | Separates styles from the component, breaking co-location.              | **Use sparingly as a powerful escape hatch.** Perfect for the 10% of cases where utilities are cumbersome. |
| **shadcn/ui**         | Building your application's UI. Buttons, forms, cards, dialogs, etc.                                 | It's a methodology, not a library. Requires setup and owning the code.  | **Adopt for your component architecture.** Build on top of it, don't just use it.                          |
| **CSS-in-JS** (e.g. Emotion) | Highly dynamic styles that depend on component props in complex ways. (Not covered in detail) | Can have a performance cost in Server Components. Less popular in App Router. | **Avoid in new Next.js App Router projects** unless you have a specific need that other methods can't solve. |

### Lessons Learned

-   **Start with a solid foundation**: Tailwind CSS provides the best-in-class foundation for styling modern Next.js applications.
-   **Combine tools strategically**: Tailwind and CSS Modules are not mutually exclusive. Use them together, leveraging the strengths of each.
-   **Build with components, not primitives**: Accelerate your development and improve quality by using a component system like shadcn/ui. Owning the code is a superpower.
-   **Design for themes from the start**: Implementing theming with CSS variables and `next-themes` is simple if you plan for it, and it dramatically improves user experience.
