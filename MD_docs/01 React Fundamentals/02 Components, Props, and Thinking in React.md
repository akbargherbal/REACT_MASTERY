# Chapter 2: Components, Props, and Thinking in React

## Function components: the only kind that matters

React components are JavaScript functions that return UI. That's it. No classes, no inheritance hierarchies, no lifecycle methods to memorize. A component is a function that takes data and returns what to display.

Before we write any code, let's understand what problem components solve. Imagine building a user interface with plain HTML and JavaScript. You'd write the same markup patterns repeatedly—a user card here, a notification badge there, a button with an icon everywhere. Components let you define these patterns once and reuse them with different data.

## The Anatomy of a Component

A React component is a function that returns JSX—a syntax that looks like HTML but is actually JavaScript. Here's the simplest possible component:

```tsx
// src/components/Welcome.tsx
function Welcome() {
  return <h1>Hello, React!</h1>;
}

export default Welcome;
```

Let's parse what's happening:

1. **Function declaration**: `function Welcome()` - Just a regular JavaScript function
2. **Return statement**: Returns JSX, which React converts to DOM elements
3. **Export**: Makes the component available to other files

JSX looks like HTML, but it's syntactic sugar for `React.createElement()` calls. When you write `<h1>Hello, React!</h1>`, React transforms it into:

```javascript
React.createElement('h1', null, 'Hello, React!')
```

You don't need to write `createElement` calls manually—that's what JSX does for you. But understanding this transformation helps when debugging.

## Using Your Component

To use a component, import it and render it like an HTML tag:

```tsx
// src/App.tsx
import Welcome from './components/Welcome';

function App() {
  return (
    <div>
      <Welcome />
      <Welcome />
      <Welcome />
    </div>
  );
}

export default App;
```

**Browser Output**:
```
Hello, React!
Hello, React!
Hello, React!
```

Each `<Welcome />` creates an independent instance of the component. They're separate pieces of UI that happen to look identical right now.

## JSX Rules You Must Know

JSX has a few non-negotiable rules:

### 1. Components must return a single root element

**This fails**:

```tsx
function Broken() {
  return (
    <h1>Title</h1>
    <p>Paragraph</p>
  );
}
```

**Terminal Output**:
```bash
src/components/Broken.tsx:3:5 - error TS1128:
Declaration or statement expected.

3     <p>Paragraph</p>
      ~
```

**Why it fails**: JSX must return one element. You're trying to return two siblings with no parent.

**Solution 1: Wrap in a div**:

```tsx
function Fixed() {
  return (
    <div>
      <h1>Title</h1>
      <p>Paragraph</p>
    </div>
  );
}
```

**Solution 2: Use a Fragment** (when you don't want an extra DOM element):

```tsx
function FixedWithFragment() {
  return (
    <>
      <h1>Title</h1>
      <p>Paragraph</p>
    </>
  );
}
```

The `<>` syntax is shorthand for `<React.Fragment>`. Fragments let you group elements without adding extra nodes to the DOM.

### 2. JSX attributes use camelCase

HTML uses `class` and `for`. JSX uses `className` and `htmlFor` because `class` and `for` are reserved JavaScript keywords.

```tsx
function StyledComponent() {
  return (
    <div className="container">
      <label htmlFor="email">Email:</label>
      <input id="email" type="email" />
    </div>
  );
}
```

### 3. JavaScript expressions go in curly braces

Any JavaScript expression can be embedded in JSX using `{}`:

```tsx
function MathComponent() {
  const x = 10;
  const y = 20;
  
  return (
    <div>
      <p>The sum is: {x + y}</p>
      <p>Random number: {Math.random()}</p>
      <p>Uppercase: {'hello'.toUpperCase()}</p>
    </div>
  );
}
```

**Browser Output**:
```
The sum is: 30
Random number: 0.7234891234
Uppercase: HELLO
```

You can put any JavaScript expression inside `{}`, but not statements. This works: `{x + y}`. This doesn't: `{if (x > 5) return y}`.

### 4. Self-closing tags must have a slash

In HTML, `<img>` and `<input>` don't need closing tags. In JSX, they must be self-closed:

```tsx
function ImageComponent() {
  return (
    <div>
      <img src="/logo.png" alt="Logo" />
      <input type="text" />
      <br />
    </div>
  );
}
```

## Component Naming Convention

Components must start with a capital letter. This is how React distinguishes components from HTML tags:

- `<button>` → HTML button element
- `<Button>` → Your custom Button component

**This fails silently**:

```tsx
function welcome() {  // lowercase - React thinks it's HTML
  return <h1>Hello</h1>;
}

function App() {
  return <welcome />;  // Renders nothing
}
```

**Browser Output**:
```
(blank screen)
```

**Browser Console**:
```
Warning: The tag <welcome> is unrecognized in this browser.
```

**Why it fails**: React sees `<welcome>` and looks for an HTML element called "welcome". There isn't one, so it renders nothing.

**Solution**: Capitalize the component name:

```tsx
function Welcome() {  // Capital W
  return <h1>Hello</h1>;
}

function App() {
  return <Welcome />;  // Works correctly
}
```

## Arrow Functions vs. Function Declarations

You can write components as arrow functions or function declarations. Both work identically:

```tsx
// Function declaration
function Welcome() {
  return <h1>Hello</h1>;
}

// Arrow function
const Welcome = () => {
  return <h1>Hello</h1>;
};

// Arrow function with implicit return
const Welcome = () => <h1>Hello</h1>;
```

Choose based on your team's preference. Function declarations are slightly more common in React codebases, but arrow functions work perfectly. The implicit return syntax (no `return` keyword, no braces) is convenient for simple components.

## Components Can Contain Logic

Components are functions, so they can contain any JavaScript logic before the return statement:

```tsx
function Greeting() {
  const hour = new Date().getHours();
  const timeOfDay = hour < 12 ? 'morning' : hour < 18 ? 'afternoon' : 'evening';
  
  return <h1>Good {timeOfDay}!</h1>;
}
```

**Browser Output** (if it's 10 AM):
```
Good morning!
```

The logic runs every time the component renders. Right now, that's once when the page loads. In Chapter 3, we'll see how to make components re-render when data changes.

## Props: passing data down

Components become useful when they can display different data. Props (short for "properties") are how you pass data into components. Think of props as function parameters—they let you customize what a component displays.

## The Problem: Hardcoded Components

Let's say you're building a user profile display. Without props, you'd need a separate component for each user:

```tsx
// src/components/UserProfileAlice.tsx
function UserProfileAlice() {
  return (
    <div className="profile">
      <h2>Alice Johnson</h2>
      <p>Software Engineer</p>
      <p>alice@example.com</p>
    </div>
  );
}

// src/components/UserProfileBob.tsx
function UserProfileBob() {
  return (
    <div className="profile">
      <h2>Bob Smith</h2>
      <p>Product Manager</p>
      <p>bob@example.com</p>
    </div>
  );
}
```

This is absurd. The structure is identical—only the data changes. Props solve this.

## Props: Function Parameters for Components

Props let you pass data to components like function arguments:

```tsx
// src/components/UserProfile.tsx
function UserProfile(props) {
  return (
    <div className="profile">
      <h2>{props.name}</h2>
      <p>{props.role}</p>
      <p>{props.email}</p>
    </div>
  );
}

export default UserProfile;
```

Now you can create multiple profiles with different data:

```tsx
// src/App.tsx
import UserProfile from './components/UserProfile';

function App() {
  return (
    <div>
      <UserProfile 
        name="Alice Johnson" 
        role="Software Engineer" 
        email="alice@example.com" 
      />
      <UserProfile 
        name="Bob Smith" 
        role="Product Manager" 
        email="bob@example.com" 
      />
    </div>
  );
}
```

**Browser Output**:
```
Alice Johnson
Software Engineer
alice@example.com

Bob Smith
Product Manager
bob@example.com
```

Each `<UserProfile />` receives different props and renders different data using the same component structure.

## Destructuring Props (The Preferred Pattern)

Instead of writing `props.name`, `props.role`, etc., destructure the props object in the function parameters:

```tsx
// src/components/UserProfile.tsx
function UserProfile({ name, role, email }) {
  return (
    <div className="profile">
      <h2>{name}</h2>
      <p>{role}</p>
      <p>{email}</p>
    </div>
  );
}
```

This is cleaner and more common in React codebases. The component works identically—destructuring is just syntactic sugar for extracting properties from the props object.

## TypeScript: Making Props Explicit

Without TypeScript, you can pass any props and React won't complain until runtime. TypeScript lets you define exactly what props a component expects:

```tsx
// src/components/UserProfile.tsx
interface UserProfileProps {
  name: string;
  role: string;
  email: string;
}

function UserProfile({ name, role, email }: UserProfileProps) {
  return (
    <div className="profile">
      <h2>{name}</h2>
      <p>{role}</p>
      <p>{email}</p>
    </div>
  );
}

export default UserProfile;
```

Now if you try to use the component incorrectly:

```tsx
// src/App.tsx
<UserProfile name="Alice" role="Engineer" />  // Missing 'email'
```

**Terminal Output**:
```bash
src/App.tsx:8:7 - error TS2741:
Property 'email' is missing in type '{ name: string; role: string; }'
but required in type 'UserProfileProps'.

8       <UserProfile name="Alice" role="Engineer" />
        ~~~~~~~~~~~~
```

TypeScript catches the error before you run the code. This is why TypeScript is valuable—it turns runtime errors into compile-time errors.

## Optional Props

Make props optional with `?`:

```tsx
interface UserProfileProps {
  name: string;
  role: string;
  email: string;
  avatarUrl?: string;  // Optional
}

function UserProfile({ name, role, email, avatarUrl }: UserProfileProps) {
  return (
    <div className="profile">
      {avatarUrl && <img src={avatarUrl} alt={name} />}
      <h2>{name}</h2>
      <p>{role}</p>
      <p>{email}</p>
    </div>
  );
}
```

The `{avatarUrl && <img ... />}` pattern means: "If `avatarUrl` exists, render the image. Otherwise, render nothing."

## Default Props

You can provide default values using JavaScript's default parameter syntax:

```tsx
interface UserProfileProps {
  name: string;
  role: string;
  email: string;
  avatarUrl?: string;
}

function UserProfile({ 
  name, 
  role, 
  email, 
  avatarUrl = '/default-avatar.png'  // Default value
}: UserProfileProps) {
  return (
    <div className="profile">
      <img src={avatarUrl} alt={name} />
      <h2>{name}</h2>
      <p>{role}</p>
      <p>{email}</p>
    </div>
  );
}
```

If `avatarUrl` isn't provided, it defaults to `'/default-avatar.png'`.

## Props Are Read-Only

**Critical rule**: Never modify props inside a component. Props flow down from parent to child and should be treated as immutable.

**This is wrong**:

```tsx
function UserProfile({ name, role, email }: UserProfileProps) {
  name = name.toUpperCase();  // ❌ Don't mutate props
  
  return (
    <div className="profile">
      <h2>{name}</h2>
      <p>{role}</p>
      <p>{email}</p>
    </div>
  );
}
```

**This is correct**:

```tsx
function UserProfile({ name, role, email }: UserProfileProps) {
  const displayName = name.toUpperCase();  // ✅ Create a new variable
  
  return (
    <div className="profile">
      <h2>{displayName}</h2>
      <p>{role}</p>
      <p>{email}</p>
    </div>
  );
}
```

Props represent data flowing into the component. If you need to transform that data, create new variables. We'll see in Chapter 3 how to handle data that changes over time.

## Passing Different Types of Props

Props can be any JavaScript value:

```tsx
interface UserCardProps {
  name: string;
  age: number;
  isActive: boolean;
  tags: string[];
  metadata: {
    joinDate: string;
    lastLogin: string;
  };
}

function UserCard({ name, age, isActive, tags, metadata }: UserCardProps) {
  return (
    <div className="user-card">
      <h2>{name}</h2>
      <p>Age: {age}</p>
      <p>Status: {isActive ? 'Active' : 'Inactive'}</p>
      <p>Tags: {tags.join(', ')}</p>
      <p>Joined: {metadata.joinDate}</p>
      <p>Last login: {metadata.lastLogin}</p>
    </div>
  );
}
```

Using it:

```tsx
<UserCard
  name="Alice"
  age={28}
  isActive={true}
  tags={['developer', 'team-lead']}
  metadata={{
    joinDate: '2023-01-15',
    lastLogin: '2024-01-10'
  }}
/>
```

Notice the double braces for objects: `metadata={{...}}`. The outer braces mean "JavaScript expression", the inner braces are the object literal.

## The Children Prop

There's a special prop called `children` that represents content between component tags:

```tsx
interface CardProps {
  title: string;
  children: React.ReactNode;
}

function Card({ title, children }: CardProps) {
  return (
    <div className="card">
      <h3>{title}</h3>
      <div className="card-content">
        {children}
      </div>
    </div>
  );
}
```

Using it:

```tsx
<Card title="User Profile">
  <p>Name: Alice</p>
  <p>Role: Engineer</p>
  <button>Edit Profile</button>
</Card>
```

Everything between `<Card>` and `</Card>` becomes the `children` prop. This pattern is fundamental to composition, which we'll explore in the next section.

`React.ReactNode` is the TypeScript type for anything that can be rendered: strings, numbers, elements, arrays of elements, or `null`/`undefined`.

## Composition over inheritance

React doesn't use inheritance. You don't extend classes or create component hierarchies. Instead, you compose components—build complex UIs by combining simple components.

This is a fundamental shift if you're coming from object-oriented frameworks. In React, composition is the only pattern you need.

## The Problem: Trying to Use Inheritance

In class-based frameworks, you might create a base `Button` class and extend it:

```
BaseButton
  ├── PrimaryButton extends BaseButton
  ├── SecondaryButton extends BaseButton
  └── DangerButton extends BaseButton
```

This creates rigid hierarchies. What if you need a button that's both primary and large? Or secondary and disabled? You end up with combinatorial explosion: `PrimaryLargeButton`, `SecondaryDisabledButton`, etc.

React solves this with composition.

## Composition Pattern 1: Props for Variants

Instead of inheritance, use props to configure behavior:

```tsx
// src/components/Button.tsx
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'danger';
  size: 'small' | 'medium' | 'large';
  children: React.ReactNode;
  onClick?: () => void;
}

function Button({ variant, size, children, onClick }: ButtonProps) {
  const baseClasses = 'px-4 py-2 rounded font-medium';
  
  const variantClasses = {
    primary: 'bg-blue-600 text-white hover:bg-blue-700',
    secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
    danger: 'bg-red-600 text-white hover:bg-red-700'
  };
  
  const sizeClasses = {
    small: 'text-sm',
    medium: 'text-base',
    large: 'text-lg'
  };
  
  const className = `${baseClasses} ${variantClasses[variant]} ${sizeClasses[size]}`;
  
  return (
    <button className={className} onClick={onClick}>
      {children}
    </button>
  );
}

export default Button;
```

Now you can create any combination:

```tsx
<Button variant="primary" size="large">Save</Button>
<Button variant="secondary" size="small">Cancel</Button>
<Button variant="danger" size="medium">Delete</Button>
```

One component, infinite variations. No inheritance needed.

## Composition Pattern 2: Container Components

Components can wrap other components to add behavior or styling:

```tsx
// src/components/Card.tsx
interface CardProps {
  children: React.ReactNode;
}

function Card({ children }: CardProps) {
  return (
    <div className="bg-white rounded-lg shadow-md p-6">
      {children}
    </div>
  );
}

// src/components/CardHeader.tsx
interface CardHeaderProps {
  children: React.ReactNode;
}

function CardHeader({ children }: CardHeaderProps) {
  return (
    <div className="border-b pb-4 mb-4">
      {children}
    </div>
  );
}

// src/components/CardContent.tsx
interface CardContentProps {
  children: React.ReactNode;
}

function CardContent({ children }: CardContentProps) {
  return (
    <div className="text-gray-700">
      {children}
    </div>
  );
}
```

Compose them together:

```tsx
<Card>
  <CardHeader>
    <h2>User Profile</h2>
  </CardHeader>
  <CardContent>
    <p>Name: Alice Johnson</p>
    <p>Role: Software Engineer</p>
  </CardContent>
</Card>
```

Each component has one job. `Card` provides the container styling. `CardHeader` adds a header section. `CardContent` styles the body. You compose them to build the complete UI.

## Composition Pattern 3: Specialized Components

Create specialized versions by wrapping generic components:

```tsx
// src/components/Button.tsx - Generic button
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'danger';
  children: React.ReactNode;
  onClick?: () => void;
}

function Button({ variant, children, onClick }: ButtonProps) {
  const variantClasses = {
    primary: 'bg-blue-600 text-white hover:bg-blue-700',
    secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
    danger: 'bg-red-600 text-white hover:bg-red-700'
  };
  
  return (
    <button 
      className={`px-4 py-2 rounded ${variantClasses[variant]}`}
      onClick={onClick}
    >
      {children}
    </button>
  );
}

// src/components/PrimaryButton.tsx - Specialized version
interface PrimaryButtonProps {
  children: React.ReactNode;
  onClick?: () => void;
}

function PrimaryButton({ children, onClick }: PrimaryButtonProps) {
  return (
    <Button variant="primary" onClick={onClick}>
      {children}
    </Button>
  );
}

// src/components/DangerButton.tsx - Another specialized version
interface DangerButtonProps {
  children: React.ReactNode;
  onClick?: () => void;
}

function DangerButton({ children, onClick }: DangerButtonProps) {
  return (
    <Button variant="danger" onClick={onClick}>
      {children}
    </Button>
  );
}
```

Now you can use either the generic `Button` with explicit variants, or the specialized versions:

```tsx
// Explicit variant
<Button variant="primary">Save</Button>

// Specialized component
<PrimaryButton>Save</PrimaryButton>
```

Both approaches work. Specialized components are convenient when you use certain configurations frequently.

## Composition Pattern 4: Render Props (Preview)

Sometimes you need to share logic between components. One pattern is passing a function as a prop that returns JSX:

```tsx
interface DataFetcherProps {
  url: string;
  render: (data: any) => React.ReactNode;
}

function DataFetcher({ url, render }: DataFetcherProps) {
  // Imagine this fetches data (we'll implement this properly in Chapter 4)
  const data = { name: 'Alice', role: 'Engineer' };
  
  return <div>{render(data)}</div>;
}
```

Using it:

```tsx
<DataFetcher
  url="/api/user"
  render={(data) => (
    <div>
      <h2>{data.name}</h2>
      <p>{data.role}</p>
    </div>
  )}
/>
```

The `DataFetcher` component handles the data fetching logic. The parent component controls how to display that data. This separates concerns: data fetching vs. presentation.

We'll see more powerful patterns for sharing logic in later chapters (custom hooks in Chapter 4, context in Chapter 11). For now, understand that composition—not inheritance—is how you build complex UIs in React.

## Why Composition Wins

**Flexibility**: Combine components in any way. No rigid hierarchies.

**Reusability**: Small, focused components can be used in many contexts.

**Maintainability**: Each component has one job. Changes are localized.

**Testability**: Small components are easier to test in isolation.

React's component model is designed around composition. Embrace it.

## Your first useful component

Let's build something real. We'll create a **User Profile Dashboard** component that displays user information, recent activity, and notifications. This will be our reference implementation—we'll evolve it through the next several chapters as we learn state management, data fetching, and more advanced patterns.

## Phase 1: The Reference Implementation

We're building a dashboard that shows:
- User profile information (name, role, email)
- Recent activity feed (list of actions)
- Notification count badge

**Project Structure**:
```
src/
├── components/
│   ├── UserDashboard.tsx      ← Our main component
│   ├── UserProfile.tsx         ← Profile section
│   ├── ActivityFeed.tsx        ← Activity list
│   └── NotificationBadge.tsx   ← Badge component
├── App.tsx
└── main.tsx
```

Let's start with the complete, working implementation using only what we've learned so far: components, props, and composition.

### The Main Dashboard Component

```tsx
// src/components/UserDashboard.tsx
import UserProfile from './UserProfile';
import ActivityFeed from './ActivityFeed';
import NotificationBadge from './NotificationBadge';

interface UserDashboardProps {
  userId: string;
}

function UserDashboard({ userId }: UserDashboardProps) {
  // For now, we'll use hardcoded data
  // In Chapter 4, we'll fetch this from an API
  const userData = {
    name: 'Alice Johnson',
    role: 'Senior Software Engineer',
    email: 'alice.johnson@example.com',
    avatarUrl: '/avatars/alice.jpg'
  };
  
  const activities = [
    { id: '1', action: 'Completed code review for PR #234', timestamp: '2 hours ago' },
    { id: '2', action: 'Deployed feature to production', timestamp: '5 hours ago' },
    { id: '3', action: 'Created new branch: feature/user-settings', timestamp: '1 day ago' }
  ];
  
  const notificationCount = 3;
  
  return (
    <div className="dashboard">
      <div className="dashboard-header">
        <h1>Dashboard</h1>
        <NotificationBadge count={notificationCount} />
      </div>
      
      <div className="dashboard-content">
        <UserProfile
          name={userData.name}
          role={userData.role}
          email={userData.email}
          avatarUrl={userData.avatarUrl}
        />
        
        <ActivityFeed activities={activities} />
      </div>
    </div>
  );
}

export default UserDashboard;
```

### The User Profile Component

```tsx
// src/components/UserProfile.tsx
interface UserProfileProps {
  name: string;
  role: string;
  email: string;
  avatarUrl: string;
}

function UserProfile({ name, role, email, avatarUrl }: UserProfileProps) {
  return (
    <div className="user-profile">
      <img 
        src={avatarUrl} 
        alt={`${name}'s avatar`}
        className="avatar"
      />
      <div className="user-info">
        <h2>{name}</h2>
        <p className="role">{role}</p>
        <p className="email">{email}</p>
      </div>
    </div>
  );
}

export default UserProfile;
```

### The Activity Feed Component

```tsx
// src/components/ActivityFeed.tsx
interface Activity {
  id: string;
  action: string;
  timestamp: string;
}

interface ActivityFeedProps {
  activities: Activity[];
}

function ActivityFeed({ activities }: ActivityFeedProps) {
  return (
    <div className="activity-feed">
      <h3>Recent Activity</h3>
      <ul className="activity-list">
        {activities.map((activity) => (
          <li key={activity.id} className="activity-item">
            <p className="activity-action">{activity.action}</p>
            <span className="activity-timestamp">{activity.timestamp}</span>
          </li>
        ))}
      </ul>
    </div>
  );
}

export default ActivityFeed;
```

### The Notification Badge Component

```tsx
// src/components/NotificationBadge.tsx
interface NotificationBadgeProps {
  count: number;
}

function NotificationBadge({ count }: NotificationBadgeProps) {
  if (count === 0) {
    return null;  // Don't render anything if no notifications
  }
  
  return (
    <div className="notification-badge">
      <span className="badge-count">{count}</span>
    </div>
  );
}

export default NotificationBadge;
```

### Using the Dashboard

```tsx
// src/App.tsx
import UserDashboard from './components/UserDashboard';

function App() {
  return (
    <div className="app">
      <UserDashboard userId="user-123" />
    </div>
  );
}

export default App;
```

### Basic Styling (Optional)

Here's minimal CSS to make it look presentable:

```css
/* src/index.css */
.dashboard {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.dashboard-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 30px;
}

.dashboard-content {
  display: grid;
  grid-template-columns: 1fr 2fr;
  gap: 20px;
}

.user-profile {
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.avatar {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  margin-bottom: 15px;
}

.user-info h2 {
  margin: 0 0 5px 0;
}

.role {
  color: #666;
  margin: 5px 0;
}

.email {
  color: #999;
  font-size: 14px;
}

.activity-feed {
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.activity-list {
  list-style: none;
  padding: 0;
}

.activity-item {
  padding: 15px 0;
  border-bottom: 1px solid #eee;
}

.activity-item:last-child {
  border-bottom: none;
}

.activity-action {
  margin: 0 0 5px 0;
}

.activity-timestamp {
  color: #999;
  font-size: 12px;
}

.notification-badge {
  background: #ef4444;
  color: white;
  border-radius: 12px;
  padding: 4px 12px;
  font-size: 14px;
  font-weight: bold;
}
```

**Browser Output**:
```
Dashboard                                    [3]

┌─────────────────────┐  ┌────────────────────────────────────┐
│  [Avatar Image]     │  │ Recent Activity                    │
│                     │  │                                    │
│  Alice Johnson      │  │ • Completed code review for PR #234│
│  Senior Software    │  │   2 hours ago                      │
│  Engineer           │  │                                    │
│  alice.johnson@     │  │ • Deployed feature to production   │
│  example.com        │  │   5 hours ago                      │
│                     │  │                                    │
│                     │  │ • Created new branch: feature/...  │
│                     │  │   1 day ago                        │
└─────────────────────┘  └────────────────────────────────────┘
```

## What We've Built

Let's analyze the component structure:

### Composition in Action

1. **UserDashboard** is the container component
   - Owns the data (for now, hardcoded)
   - Passes data down to child components via props
   - Handles layout and structure

2. **UserProfile** is a presentational component
   - Receives user data via props
   - Displays that data
   - Has no knowledge of where the data comes from

3. **ActivityFeed** is a list component
   - Receives an array of activities
   - Maps over the array to render individual items
   - Uses the `key` prop (we'll explain why in Chapter 5)

4. **NotificationBadge** is a conditional component
   - Returns `null` if count is 0 (renders nothing)
   - Otherwise displays the count

### Data Flow

Data flows in one direction: **down**.

```
UserDashboard (owns data)
    ↓ props
    ├─→ UserProfile (displays user data)
    ├─→ ActivityFeed (displays activities)
    └─→ NotificationBadge (displays count)
```

Parent components pass data to children via props. Children cannot modify that data—they can only display it.

### TypeScript Benefits

Notice how TypeScript helps:

1. **Interface definitions** document what each component expects
2. **Type checking** prevents passing wrong data types
3. **Autocomplete** in your editor shows available props
4. **Refactoring safety** - if you change a prop name, TypeScript shows all places that need updating

## Current Limitations (Preview of What's Coming)

This dashboard works, but it has significant limitations:

1. **Static data**: Everything is hardcoded. In Chapter 4, we'll fetch real data from an API.

2. **No interactivity**: You can't click anything or update the UI. In Chapter 3, we'll add state and event handlers.

3. **No loading states**: When we fetch data, users will see a blank screen. In Chapter 4, we'll add loading indicators.

4. **No error handling**: If data fetching fails, the app crashes. In Chapter 4, we'll handle errors gracefully.

5. **Performance issues**: The entire dashboard re-renders even if only one piece changes. In Chapter 5, we'll optimize this.

But right now, we have a solid foundation. We've built a component hierarchy using composition, passed data via props, and created a maintainable structure.

## Key Takeaways

### Components Are Functions

React components are JavaScript functions that return JSX. No classes, no inheritance, just functions.

### Props Flow Down

Data flows from parent to child via props. Props are read-only—never modify them inside a component.

### Composition Over Inheritance

Build complex UIs by combining simple components. Use props for variants, container components for layout, and specialized components for common patterns.

### TypeScript Adds Safety

Define prop interfaces to catch errors at compile time. TypeScript makes refactoring safer and provides better developer experience.

### One Component, One Job

Each component should have a single responsibility. `UserProfile` displays user info. `ActivityFeed` displays activities. `UserDashboard` coordinates them.

## What's Next

In Chapter 3, we'll make this dashboard interactive. We'll add:
- State management with `useState`
- Event handlers for user interactions
- The ability to update the UI in response to user actions

Our static dashboard will become dynamic. But first, we need to understand why regular JavaScript variables don't work in React—and what React does instead.
