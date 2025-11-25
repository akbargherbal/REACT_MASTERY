# Chapter 2: Components, Props, and Thinking in React

## Function components: the only kind that matters

## The Building Block of React: The Component

Everything in a React application is a component. A button is a component. A form is a component. An entire page is a component. A component is a self-contained, reusable piece of UI. The core idea of React is to break down a complex user interface into a tree of these small, manageable components.

In modern React, a component is simply a JavaScript function.

That's it. It's a function that accepts some data as an argument and returns a description of what the UI should look like. This "description" is written in a syntax called JSX (JavaScript XML), which looks very much like HTML.

Let's look at the simplest possible React component.

```tsx
// src/components/Welcome.tsx

function Welcome() {
  return <h1>Hello, world!</h1>;
}

export default Welcome;
```

This `Welcome` function is a valid React component. It takes no arguments and returns an `h1` element. The browser doesn't understand JSX directly; your build tool (like the one included in Next.js) transforms this HTML-like syntax into regular JavaScript that creates the corresponding DOM elements.

### A Quick Note on History: Class Components

If you look at older React tutorials or codebases, you might see components written as JavaScript classes:

```jsx
// Old way: Class Component (for historical context only)
import React from 'react';

class Welcome extends React.Component {
  render() {
    return <h1>Hello, world!</h1>;
  }
}
```

For many years, this was the standard way to write components, especially those that needed to manage state or lifecycle events. However, with the introduction of **Hooks** in React 16.8 (which we'll cover in detail in Chapter 3), function components became capable of everything class components could do, but with a much simpler and more direct syntax.

In this book, and in modern professional React development, **we will exclusively use function components**. They are simpler to write, easier to read, and the official recommendation from the React team for all new development. Understanding that class components exist is useful for historical context, but you do not need to learn how to write them.

## Your first useful component

## Phase 1: Establish the Reference Implementation

To learn how to "think in React," we need a concrete example to work with. Abstract concepts are hard to grasp, so we'll build a real UI element and improve it step-by-step throughout this chapter.

Our anchor example will be a **`UserProfileCard`**. It's a common UI pattern that's simple enough to understand but complex enough to reveal the core principles of component design.

Let's start with the most direct, naive implementation. We'll create a single component and hardcode all the information directly into it.

**Project Structure**:
```
src/
└── app/
    └── page.tsx
└── components/
    └── UserProfileCard.tsx  ← We will create this file
```

Here is our first version, our reference implementation. It works, but as we'll soon see, it's deeply flawed.

```tsx
// src/components/UserProfileCard.tsx

export default function UserProfileCard() {
  const userName = "Ada Lovelace";
  const userHandle = "@ada";
  const bio = "First computer programmer. Enchantress of Numbers.";
  const avatarUrl = "https://i.imgur.com/Q9qg4MC.jpeg"; // A placeholder image

  return (
    <div style={{ 
      border: '1px solid #ccc', 
      borderRadius: '8px', 
      padding: '16px', 
      maxWidth: '300px',
      fontFamily: 'sans-serif'
    }}>
      <img 
        src={avatarUrl} 
        alt={`${userName}'s avatar`}
        style={{ width: '100px', height: '100px', borderRadius: '50%' }} 
      />
      <h2 style={{ margin: '10px 0 5px' }}>{userName}</h2>
      <p style={{ color: '#666', margin: '0 0 10px' }}>{userHandle}</p>
      <p>{bio}</p>
    </div>
  );
}
```

To see this on the screen, we'll import it into our main page file.

```tsx
// src/app/page.tsx

import UserProfileCard from '@/components/UserProfileCard';

export default function HomePage() {
  return (
    <main style={{ display: 'flex', justifyContent: 'center', paddingTop: '40px' }}>
      <UserProfileCard />
    </main>
  );
}
```

### Iteration 0: The Monolithic Component

When you run this, you'll see a nicely formatted user profile card for Ada Lovelace. It works perfectly.

**Browser Behavior**:
A single user profile card for Ada Lovelace is displayed on the page.

So, what's the problem? The problem isn't a bug the user can see; it's a structural problem that affects us, the developers.

**Current Limitation**: This component is completely rigid.
1.  **It's not reusable**: It will *only* ever display information for Ada Lovelace. What if we want to show a card for Grace Hopper? Or Alan Turing?
2.  **It's not configurable**: The data is hardcoded inside the component. To change the user, you have to edit the component's source code.
3.  **It's not composable**: It's one giant block of JSX. We can't reuse just the avatar part or just the user info part in another place in our application.

This monolithic approach is a dead end. In the next section, we'll address the first and most critical flaw: its lack of reusability.

## Props: passing data down

## Iteration 1: Making the Component Reusable with Props

Our `UserProfileCard` works, but it's a one-trick pony. Let's introduce a new scenario that immediately breaks our current approach.

**New Scenario**: The product manager asks us to display a list of three different users on the homepage.

With our current `UserProfileCard.tsx`, our only option is to copy and paste the entire component's code three times, changing the hardcoded values in each copy. This is a recipe for disaster. A small change to the card's layout would require editing it in three separate places.

This is a **maintainability failure**. We need a way to separate the component's structure (the JSX) from the data it displays.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The user sees exactly what we coded: one card for Ada Lovelace. The problem isn't what the user sees, but the pain we experience as developers trying to meet the new requirement.

**Code Smell Evidence**:
- **Hardcoded Values**: `const userName = "Ada Lovelace";` couples the data directly to the presentation.
- **Low Cohesion**: The component is responsible for both its structure *and* its specific data.
- **No Reusability**: The component cannot be used in any context other than displaying Ada Lovelace.

**Let's parse this evidence**:

1.  **What the developer experiences**: To show three users, I have to create `UserProfileCardAda.tsx`, `UserProfileCardGrace.tsx`, and `UserProfileCardAlan.tsx`. This is tedious and error-prone.

2.  **What the code reveals**: The component's logic is not portable. The data is "stuck" inside.

3.  **Root cause identified**: The component is self-contained in the wrong way. It fetches or defines its own data, making it impossible to configure from the outside.

4.  **Why the current approach can't solve this**: It doesn't scale. For 100 users, we would need 100 component files. This is fundamentally unworkable.

5.  **What we need**: A mechanism to pass data *into* a component from its parent, just like passing arguments to a function.

### Technique Introduced: Props

In React, this mechanism is called **props** (short for "properties"). Props are to components what arguments are to functions. They are the primary way to pass data from a parent component down to a child component.

Props are passed as attributes in JSX, just like HTML attributes. The child component receives them as a single object, which is the first argument to its function.

To make this robust, we'll use TypeScript to define the "shape" of our props object. This creates a contract: anyone using `UserProfileCard` *must* provide these specific props with the correct data types.

### Solution Implementation

Let's refactor `UserProfileCard` to accept its data via props.

**Before** (Iteration 0): Hardcoded data inside the component.

```tsx
// src/components/UserProfileCard.tsx (Iteration 0)

export default function UserProfileCard() {
  const userName = "Ada Lovelace";
  const userHandle = "@ada";
  const bio = "First computer programmer. Enchantress of Numbers.";
  const avatarUrl = "https://i.imgur.com/Q9qg4MC.jpeg";

  return (
    <div /* ... styles ... */>
      <img src={avatarUrl} /* ... */ />
      <h2>{userName}</h2>
      <p>{userHandle}</p>
      <p>{bio}</p>
    </div>
  );
}
```

**After** (Iteration 1): Data is received via props.

```tsx
// src/components/UserProfileCard.tsx (Iteration 1)

// 1. Define the shape of the props object with a TypeScript interface
interface UserProfileCardProps {
  userName: string;
  userHandle: string;
  bio: string;
  avatarUrl: string;
}

// 2. The component function now accepts a 'props' object
export default function UserProfileCard(props: UserProfileCardProps) {
  // 3. Destructure the props object for easier access
  const { userName, userHandle, bio, avatarUrl } = props;

  return (
    <div style={{ 
      border: '1px solid #ccc', 
      borderRadius: '8px', 
      padding: '16px', 
      maxWidth: '300px',
      fontFamily: 'sans-serif'
    }}>
      <img 
        src={avatarUrl} 
        alt={`${userName}'s avatar`}
        style={{ width: '100px', height: '100px', borderRadius: '50%' }} 
      />
      <h2 style={{ margin: '10px 0 5px' }}>{userName}</h2>
      <p style={{ color: '#666', margin: '0 0 10px' }}>{userHandle}</p>
      <p>{bio}</p>
    </div>
  );
}
```

Our component is now a pure, reusable template. It describes how to render a user profile, but it has no knowledge of *which* user it's rendering. That responsibility now belongs to the parent component.

### Verification

Let's update our homepage to render three different user profiles using our new, flexible component.

```tsx
// src/app/page.tsx

import UserProfileCard from '@/components/UserProfileCard';

const users = [
  {
    userName: "Ada Lovelace",
    userHandle: "@ada",
    bio: "First computer programmer. Enchantress of Numbers.",
    avatarUrl: "https://i.imgur.com/Q9qg4MC.jpeg",
  },
  {
    userName: "Grace Hopper",
    userHandle: "@grace",
    bio: "Pioneering computer scientist and US Navy rear admiral.",
    avatarUrl: "https://i.imgur.com/jA8hHMpm.jpg",
  },
  {
    userName: "Alan Turing",
    userHandle: "@alan",
    bio: "Father of theoretical computer science and artificial intelligence.",
    avatarUrl: "https://i.imgur.com/4oTq0K8.jpg",
  }
];

export default function HomePage() {
  return (
    <main style={{ 
      display: 'flex', 
      gap: '20px', 
      justifyContent: 'center', 
      paddingTop: '40px' 
    }}>
      {users.map(user => (
        <UserProfileCard
          key={user.userHandle} // 'key' is a special prop for lists, covered later
          userName={user.userName}
          userHandle={user.userHandle}
          bio={user.bio}
          avatarUrl={user.avatarUrl}
        />
      ))}
    </main>
  );
}
```

**Expected vs. Actual Improvement**:
- **Expected**: We should be able to render multiple, different user cards without duplicating component code.
- **Actual**: The browser now displays three distinct user profile cards, side-by-side. We achieved this by reusing a single component definition, which is a huge win for maintainability.

**Limitation Preview**: Our component is reusable, but it's still monolithic. The layout is rigid. What if we want to add a "Verified" badge for one user, or extra action buttons for another? Adding more and more props to handle every variation will quickly become messy. This leads us to the next core principle of React: composition.

### Common Failure Modes and Their Signatures

#### Symptom: "Property 'userName' is missing in type '{}' but required in type 'UserProfileCardProps'."

**Terminal Output**:
```bash
src/app/page.tsx:35:9 - error TS2741:
Property 'userName' is missing in type '{}' but required in type 'UserProfileCardProps'.

35         <UserProfileCard />
           ~~~~~~~~~~~~~~~~~~~
```

**Root cause**: You are trying to render the component without passing the required props. Our TypeScript interface `UserProfileCardProps` created a contract, and we violated it.
**Solution**: Ensure you pass all props defined as required in the component's prop types.

#### Symptom: The component displays a value as `undefined` or an image is broken.

**Browser behavior**: The card renders, but the `h2` for the name is blank, or the `img` tag has no `src`.

**React DevTools Clues**:
- Select the `UserProfileCard` component in the Components tab.
- Look at the `props` panel on the right.
- You might see `userName: undefined` or notice a typo like `userNmae: "Ada Lovelace"`.

**Root cause**: You likely made a typo in the prop name when rendering the component (e.g., `userName` vs. `userNmae`). React doesn't know you meant `userName`, so the prop is never received by the child component.
**Solution**: Check for typos in your prop names where you call the component. This is where TypeScript helps immensely, as it would have caught the typo before you even ran the code.

## Composition over inheritance

## Iteration 2: Making the Component Composable

Our `UserProfileCard` is now reusable thanks to props. But it's still a rigid block. Let's introduce a new requirement that highlights this inflexibility.

**New Scenario**: The design team wants two new variations of the user card:
1.  A "Premium" user card that has a golden border and a "Premium Member" badge inside.
2.  An "Admin" user card that includes "Edit" and "Delete" buttons at the bottom.

The naive approach is to add more props to our `UserProfileCard`: `isPremium: boolean`, `isAdmin: boolean`.

Let's see what that looks like.

```tsx
// AVOID THIS PATTERN: UserProfileCard with boolean props for layout changes

interface UserProfileCardProps {
  // ... existing props
  isPremium?: boolean;
  isAdmin?: boolean;
}

export default function UserProfileCard(props: UserProfileCardProps) {
  const { userName, userHandle, bio, avatarUrl, isPremium, isAdmin } = props;

  const cardStyle = {
    border: isPremium ? '2px solid gold' : '1px solid #ccc', // Conditional style
    // ... other styles
  };

  return (
    <div style={cardStyle}>
      {/* ... avatar, name, bio ... */}
      {isPremium && <p style={{color: 'gold'}}>Premium Member</p>} {/* Conditional UI */}
      {isAdmin && (
        <div>
          <button>Edit</button>
          <button>Delete</button>
        </div>
      )}
    </div>
  );
}
```

This works, but it's a path to chaos. What happens when we need a `isDeactivated` state? Or a `hasNewMessage` indicator? The component becomes a tangled mess of conditional logic. This is a **complexity failure**.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**: The UI works as requested. We can create premium and admin cards. The failure is in the code's architecture.

**Code Smell Evidence**:
- **High Cyclomatic Complexity**: The number of possible paths through the component's render logic is growing.
- **Violates Single Responsibility Principle**: The component is now responsible for its own layout AND the layout of premium badges AND the layout of admin buttons.
- **Fragile**: Adding a new variation requires modifying and re-testing the core `UserProfileCard`, risking regressions for all other variations.

**Let's parse this evidence**:

1.  **What the developer experiences**: The component is becoming hard to read and reason about. I have to mentally track all the boolean props to understand what the final output will be.

2.  **What the code reveals**: The component is not "open for extension but closed for modification." To extend its functionality, we have to modify its internal logic.

3.  **Root cause identified**: The component is trying to own and control every possible piece of UI that could ever appear inside it. This is thinking in terms of **inheritance** (a PremiumCard *is a* UserProfileCard with modifications) rather than **composition**.

4.  **Why the current approach can't solve this**: It's not scalable. Each new UI variation adds another layer of conditional logic, leading to an exponential increase in complexity.

5.  **What we need**: A way for the parent component to decide what extra UI, if any, gets rendered inside the card, without the card needing to know anything about it.

### Technique Introduced: Composition and the `children` Prop

The React philosophy is **composition over inheritance**. Instead of creating a single, monolithic component that handles all variations, we should create smaller, focused components and assemble them like Lego bricks.

The key to this is a special prop called `children`. Whatever JSX you put *between* a component's opening and closing tags is passed to that component as the `children` prop.

Let's create a generic `Card` component that knows nothing about users. It only knows how to draw a box.

```tsx
// src/components/Card.tsx (New file)
import React from 'react';

interface CardProps {
  children: React.ReactNode; // React.ReactNode can be any valid JSX
  borderColor?: string;
}

export default function Card({ children, borderColor = '#ccc' }: CardProps) {
  return (
    <div style={{
      border: `1px solid ${borderColor}`,
      borderRadius: '8px',
      padding: '16px',
      maxWidth: '300px',
      fontFamily: 'sans-serif'
    }}>
      {children} {/* This is where the magic happens */}
    </div>
  );
}
```

This `Card` component is beautifully simple and infinitely reusable. It takes any content (`children`) and wraps it in a styled `div`.

### Solution Implementation

Now, let's break our original `UserProfileCard` into smaller pieces and reassemble them using our new `Card` component.

**Step 1: Create smaller, focused components.**

```tsx
// src/components/Avatar.tsx (New file)
interface AvatarProps {
  src: string;
  alt: string;
}
export default function Avatar({ src, alt }: AvatarProps) {
  return (
    <img 
      src={src} 
      alt={alt}
      style={{ width: '100px', height: '100px', borderRadius: '50%' }} 
    />
  );
}

// src/components/UserInfo.tsx (New file)
interface UserInfoProps {
  name: string;
  handle: string;
  bio: string;
}
export default function UserInfo({ name, handle, bio }: UserInfoProps) {
  return (
    <>
      <h2 style={{ margin: '10px 0 5px' }}>{name}</h2>
      <p style={{ color: '#666', margin: '0 0 10px' }}>{handle}</p>
      <p>{bio}</p>
    </>
  );
}
```

Notice these components are purely presentational. They take data and return UI. They have no wrappers or outer divs.

**Step 2: Re-compose the `UserProfileCard` from these pieces.**

Our old `UserProfileCard.tsx` file is no longer needed. The "user profile card" is not a single component anymore; it's a *pattern of composition* that we create in our page file.

### Verification

Let's update `page.tsx` to build the standard, premium, and admin cards by composing our new, small components.

```tsx
// src/app/page.tsx (Final Version)

import Card from '@/components/Card';
import Avatar from '@/components/Avatar';
import UserInfo from '@/components/UserInfo';

const ada = {
  userName: "Ada Lovelace",
  userHandle: "@ada",
  bio: "First computer programmer. Enchantress of Numbers.",
  avatarUrl: "https://i.imgur.com/Q9qg4MC.jpeg",
};

const grace = {
  userName: "Grace Hopper",
  userHandle: "@grace",
  bio: "Pioneering computer scientist and US Navy rear admiral.",
  avatarUrl: "https://i.imgur.com/jA8hHMpm.jpg",
};

export default function HomePage() {
  return (
    <main style={{ 
      display: 'flex', 
      gap: '20px', 
      justifyContent: 'center', 
      alignItems: 'flex-start',
      padding: '40px' 
    }}>
      {/* Standard User Card */}
      <Card>
        <Avatar src={ada.avatarUrl} alt={ada.userName} />
        <UserInfo name={ada.userName} handle={ada.userHandle} bio={ada.bio} />
      </Card>

      {/* Premium User Card */}
      <Card borderColor="gold">
        <Avatar src={grace.avatarUrl} alt={grace.userName} />
        <UserInfo name={grace.userName} handle={grace.userHandle} bio={grace.bio} />
        <p style={{ color: 'gold', fontWeight: 'bold', marginTop: '10px' }}>
          Premium Member
        </p>
      </Card>

      {/* Admin User Card (using Ada's data for example) */}
      <Card>
        <Avatar src={ada.avatarUrl} alt={ada.userName} />
        <UserInfo name={ada.userName} handle={ada.userHandle} bio={ada.bio} />
        <div style={{ marginTop: '15px', display: 'flex', gap: '10px' }}>
          <button>Edit Profile</button>
          <button>Delete User</button>
        </div>
      </Card>
    </main>
  );
}
```

**Expected vs. Actual Improvement**:
- **Expected**: We should be able to create different card layouts without modifying any of the core components (`Card`, `Avatar`, `UserInfo`).
- **Actual**: We did exactly that. The parent component (`HomePage`) now has full control over the layout. The `Card` component doesn't know or care that it contains admin buttons or premium badges. This is the power of composition. Our code is now more flexible, reusable, and easier to maintain.

### Debugging Workflow: When Your Component Fails

**Step 1: Observe the user experience**
You create a new card, but the admin buttons you added don't appear.

**Step 2: Check the console**
There are no errors. This is a logic issue, not a crash.

**Step 3: Inspect with React DevTools**
- Open the Components tab in your browser's DevTools.
- Find and click on your `<Card>` component in the component tree.
- Look at the `props` panel on the right. You will see a `children` prop.
- You can expand `children`. If it's an array, you can inspect each child. You should see your `Avatar`, `UserInfo`, and the `div` with your buttons. If they are there, the problem is not with the parent passing them down.
- Now, look at the source code for `Card.tsx`. Is `{children}` actually being rendered? A common mistake is to define a component that accepts children but forget to render them.

This systematic inspection allows you to trace the flow of your UI from parent to child and pinpoint exactly where the breakdown occurs.

## Synthesis: The Complete Journey

## The Journey: From Problem to Solution

We have transformed our code from a rigid, unmaintainable block into a flexible, reusable system of components. This journey reflects the core of "thinking in React."

| Iteration | Failure Mode                               | Technique Applied      | Result                                                              | Code Quality Impact                               |
| :-------- | :----------------------------------------- | :--------------------- | :------------------------------------------------------------------ | :------------------------------------------------ |
| 0         | **Rigidity & Lack of Reusability**: Hardcoded data. | None (Initial State)   | A single, non-reusable card for one user.                           | Brittle, high maintenance, not scalable.          |
| 1         | **Maintainability Failure**: Copy-pasting code to show new users. | **Props** & TypeScript Interfaces | A single, reusable `UserProfileCard` component configurable with data. | Reusable, maintainable, type-safe.                |
| 2         | **Complexity Failure**: Adding boolean props and conditional logic for UI variations. | **Composition** (`children` prop) & Component Splitting | Small, single-purpose components (`Card`, `Avatar`, `UserInfo`) composed by the parent. | Flexible, scalable, follows Single Responsibility Principle. |

### Final Implementation

Our final project structure reflects this compositional approach:

**Project Structure**:
```
src/
└── app/
    └── page.tsx              ← Composition happens here
└── components/
    ├── Avatar.tsx            ← Reusable UI piece
    ├── Card.tsx              ← Reusable container
    └── UserInfo.tsx          ← Reusable UI piece
```

The final code in `page.tsx` is declarative. It reads like a description of the UI we want to build, assembling our custom "Lego bricks" to achieve the desired result.

### Decision Framework: Props vs. Children

When building a component, you'll often face a choice: should this be controlled by a prop, or should I use `children`?

| Scenario                               | Choose          | Why?                                                                                              | Example                                                                |
| :------------------------------------- | :-------------- | :------------------------------------------------------------------------------------------------ | :--------------------------------------------------------------------- |
| Passing simple data (strings, numbers, booleans) | **Props**       | Props are explicit and create a clear API for your component. They are easy to type-check.        | `<Avatar src="..." alt="..." />`                                       |
| Passing a data object                  | **Props**       | The component needs the data to render its internal structure.                                    | `<UserInfo user={userData} />`                                         |
| Customizing a component's visual variant | **Props**       | For a small, fixed set of variations, props are simpler.                                          | `<Button variant="primary" />` vs. `<Button variant="secondary" />`     |
| Passing arbitrary or unknown content   | **`children`**  | The component is a container and shouldn't care what's inside it.                                 | `<Card>...any content here...</Card>`                                   |
| Creating different layouts with the same "chrome" | **`children`**  | Provides maximum flexibility to the parent component to control the layout.                       | Our Admin Card vs. Premium Card example.                               |
| Wrapping other components              | **`children`**  | The core pattern for Higher-Order Components and providers (which we'll see later).               | `<AuthProvider>{/* The rest of the app */}</AuthProvider>`             |

**Rule of Thumb**: Use props for *what* a component is (its data, its identity). Use `children` for *what's inside* it (its contents, its layout).

### Lessons Learned

1.  **Start by Building, then Decompose**: It's often easiest to build a static, monolithic version of your UI first, as we did in Iteration 0. Once it looks right, break it down into a hierarchy of reusable components.
2.  **Unidirectional Data Flow**: Notice that data always flows down: from the parent (`HomePage`) to the children (`Card`, `Avatar`). A child component never modifies the props it receives. This is a fundamental principle in React that makes applications easier to reason about.
3.  **Composition is King**: Favoring composition over prop-based conditional logic leads to smaller, more predictable, and more reusable components. Your future self (and your teammates) will thank you for it.
