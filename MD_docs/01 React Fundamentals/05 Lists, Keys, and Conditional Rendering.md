# Chapter 5: Lists, Keys, and Conditional Rendering

## Rendering arrays efficiently

## Rendering Arrays Efficiently

In Chapter 4, we built a `UserDashboard` that fetches and displays user data. Now we need to extend it to show a list of activities—recent actions the user has taken. This is where React's array rendering capabilities come into play.

But before we dive into the "right way," let's see what happens when we approach this naively.

### Phase 1: The Reference Implementation

We're building an **Activity Feed** component that displays a user's recent actions. This will be our anchor example throughout this chapter, evolving through multiple iterations as we discover and fix problems.

**Project Structure**:
```
src/
├── components/
│   ├── UserDashboard.tsx
│   ├── ActivityFeed.tsx      ← Our new reference implementation
│   └── ActivityItem.tsx
├── types/
│   └── activity.ts
└── app/
    └── page.tsx
```

Let's start with the data structure:

```typescript
// src/types/activity.ts
export interface Activity {
  id: string;
  type: 'login' | 'purchase' | 'comment' | 'like' | 'share';
  description: string;
  timestamp: Date;
  metadata?: {
    amount?: number;
    productName?: string;
    targetUser?: string;
  };
}
```

Now, our first attempt at rendering this list:

```tsx
// src/components/ActivityFeed.tsx - Version 1 (Naive Implementation)
import { useState, useEffect } from 'react';
import { Activity } from '../types/activity';

export function ActivityFeed({ userId }: { userId: string }) {
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/users/${userId}/activities`)
      .then(res => res.json())
      .then(data => {
        setActivities(data);
        setIsLoading(false);
      });
  }, [userId]);

  if (isLoading) {
    return <div>Loading activities...</div>;
  }

  return (
    <div className="activity-feed">
      <h2>Recent Activity</h2>
      <div className="activity-list">
        {activities.map(activity => (
          <div className="activity-item">
            <span className="activity-type">{activity.type}</span>
            <span className="activity-description">{activity.description}</span>
            <span className="activity-time">
              {new Date(activity.timestamp).toLocaleString()}
            </span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

This code runs without errors. It displays the activities. It seems to work perfectly.

**But there's a hidden problem.**

### The Failure: React's Warning in the Console

Let's run this component with real data and check the browser console.

**Browser Console**:
```
Warning: Each child in a list should have a unique "key" prop.

Check the render method of `ActivityFeed`. See https://reactjs.org/link/warning-keys for more information.
    at div
    at ActivityFeed
```

**Browser Behavior**:
The activities display correctly. No visual problems. The warning is easy to ignore—after all, everything *looks* fine.

### Diagnostic Analysis: Why React Complains

**What the console reveals**:
React is warning us about a missing `key` prop. This isn't a hard error—the app still works. But React is telling us we're doing something that will cause problems.

**What's actually happening under the hood**:
When React renders a list, it needs to track which items are which across re-renders. Without keys, React uses the array index as an implicit key. This works fine... until it doesn't.

**Why the current approach can't scale**:
Imagine we add the ability to:
- Delete an activity
- Add new activities in real-time
- Reorder activities by timestamp
- Filter activities by type

Without proper keys, React will struggle to efficiently update the list. It might re-render items unnecessarily, lose component state, or even display the wrong data.

**What we need**: A way to give React a stable identity for each list item.

### The Concept: React's Reconciliation Algorithm

Before we fix the code, understand what React is doing when it renders a list.

**The Problem React Solves**:
When your component re-renders with a new array, React needs to figure out:
1. Which items are new (add them to the DOM)
2. Which items were removed (remove them from the DOM)
3. Which items moved (reorder them in the DOM)
4. Which items changed (update them in the DOM)

**Without Keys** (using array index):
```
Old array: [A, B, C]  →  Indices: [0, 1, 2]
New array: [A, C, D]  →  Indices: [0, 1, 2]

React thinks:
- Index 0: A → A (no change)
- Index 1: B → C (update B to look like C)
- Index 2: C → D (update C to look like D)
```

React doesn't realize that B was deleted and D was added. It thinks B and C changed. This causes unnecessary re-renders and can lose component state.

**With Keys** (using unique IDs):
```
Old array: [A(id:1), B(id:2), C(id:3)]
New array: [A(id:1), C(id:3), D(id:4)]

React thinks:
- id:1 (A): Still here, no change
- id:2 (B): Gone, remove it
- id:3 (C): Still here, moved from index 2 to index 1
- id:4 (D): New, add it
```

React correctly identifies what changed. It can reuse existing DOM nodes efficiently.

### Solution: Adding Keys

The fix is simple—add a `key` prop to each list item:

```tsx
// src/components/ActivityFeed.tsx - Version 2 (With Keys)
import { useState, useEffect } from 'react';
import { Activity } from '../types/activity';

export function ActivityFeed({ userId }: { userId: string }) {
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/users/${userId}/activities`)
      .then(res => res.json())
      .then(data => {
        setActivities(data);
        setIsLoading(false);
      });
  }, [userId]);

  if (isLoading) {
    return <div>Loading activities...</div>;
  }

  return (
    <div className="activity-feed">
      <h2>Recent Activity</h2>
      <div className="activity-list">
        {activities.map(activity => (
          <div key={activity.id} className="activity-item">
            <span className="activity-type">{activity.type}</span>
            <span className="activity-description">{activity.description}</span>
            <span className="activity-time">
              {new Date(activity.timestamp).toLocaleString()}
            </span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**What changed**:
- Added `key={activity.id}` to the mapped `div`
- Used the activity's unique `id` field as the key

**Browser Console**:
```
(No warnings)
```

**Verification**:
The warning is gone. React can now efficiently track each activity item.

### Extracting to a Component

As our activity items get more complex, let's extract them into a separate component. This is a common pattern—it keeps the list rendering logic clean and makes each item easier to test and maintain.

```tsx
// src/components/ActivityItem.tsx
import { Activity } from '../types/activity';

interface ActivityItemProps {
  activity: Activity;
}

export function ActivityItem({ activity }: ActivityItemProps) {
  const formatTimestamp = (date: Date) => {
    const now = new Date();
    const diffMs = now.getTime() - new Date(date).getTime();
    const diffMins = Math.floor(diffMs / 60000);
    
    if (diffMins < 1) return 'Just now';
    if (diffMins < 60) return `${diffMins}m ago`;
    if (diffMins < 1440) return `${Math.floor(diffMins / 60)}h ago`;
    return new Date(date).toLocaleDateString();
  };

  const getActivityIcon = (type: Activity['type']) => {
    const icons = {
      login: '🔐',
      purchase: '🛒',
      comment: '💬',
      like: '❤️',
      share: '🔄'
    };
    return icons[type];
  };

  return (
    <div className="activity-item">
      <span className="activity-icon">{getActivityIcon(activity.type)}</span>
      <div className="activity-content">
        <p className="activity-description">{activity.description}</p>
        {activity.metadata?.productName && (
          <span className="activity-meta">
            Product: {activity.metadata.productName}
          </span>
        )}
        {activity.metadata?.amount && (
          <span className="activity-meta">
            Amount: ${activity.metadata.amount}
          </span>
        )}
      </div>
      <span className="activity-time">{formatTimestamp(activity.timestamp)}</span>
    </div>
  );
}
```

```tsx
// src/components/ActivityFeed.tsx - Version 3 (With Extracted Component)
import { useState, useEffect } from 'react';
import { Activity } from '../types/activity';
import { ActivityItem } from './ActivityItem';

export function ActivityFeed({ userId }: { userId: string }) {
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/users/${userId}/activities`)
      .then(res => res.json())
      .then(data => {
        setActivities(data);
        setIsLoading(false);
      });
  }, [userId]);

  if (isLoading) {
    return <div>Loading activities...</div>;
  }

  return (
    <div className="activity-feed">
      <h2>Recent Activity</h2>
      <div className="activity-list">
        {activities.map(activity => (
          <ActivityItem key={activity.id} activity={activity} />
        ))}
      </div>
    </div>
  );
}
```

**Important**: The `key` prop stays on the component in the `.map()` call, not inside the `ActivityItem` component itself. Keys are used by React's reconciliation algorithm and are not passed as props to your component.

### When You Don't Have a Unique ID

Sometimes your data doesn't come with unique IDs. What then?

**Option 1: Generate IDs on the server** (preferred)
If you control the API, add unique IDs to your data structure.

**Option 2: Generate stable IDs on the client**
If the data is truly static (never changes), you can generate IDs once:

```typescript
// Generate IDs once when data arrives
const activitiesWithIds = rawActivities.map((activity, index) => ({
  ...activity,
  id: `${activity.type}-${activity.timestamp}-${index}`
}));
```

**Option 3: Use array index as a last resort**
Only if:
- The list never reorders
- Items are never added/removed from the middle
- Items don't have any internal state

```tsx
{activities.map((activity, index) => (
  <ActivityItem key={index} activity={activity} />
))}
```

**Warning**: Using index as key is almost always wrong. It defeats the purpose of keys and can cause subtle bugs.

### Limitation Preview

Our activity feed now renders efficiently with proper keys. But we still have problems:

1. **No real-time updates**: New activities don't appear automatically
2. **No filtering**: Users can't filter by activity type
3. **Performance**: What happens with 1,000 activities?

We'll address these in the next sections.

## Why keys matter (and what happens when you get them wrong)

## Why Keys Matter (And What Happens When You Get Them Wrong)

We added keys to fix a warning. But let's see what actually breaks when keys are wrong. This section will demonstrate the concrete failures that occur with improper key usage.

### Iteration 1: Adding Interactive State

Let's make our activity feed interactive. Users can now "like" activities, and we'll track which ones they've liked.

```tsx
// src/components/ActivityItem.tsx - Version 2 (With Like Button)
import { useState } from 'react';
import { Activity } from '../types/activity';

interface ActivityItemProps {
  activity: Activity;
  onLike: (activityId: string) => void;
}

export function ActivityItem({ activity, onLike }: ActivityItemProps) {
  const [isLiked, setIsLiked] = useState(false);
  const [likeCount, setLikeCount] = useState(activity.metadata?.likes || 0);

  const handleLike = () => {
    setIsLiked(!isLiked);
    setLikeCount(prev => isLiked ? prev - 1 : prev + 1);
    onLike(activity.id);
  };

  const formatTimestamp = (date: Date) => {
    const now = new Date();
    const diffMs = now.getTime() - new Date(date).getTime();
    const diffMins = Math.floor(diffMs / 60000);
    
    if (diffMins < 1) return 'Just now';
    if (diffMins < 60) return `${diffMins}m ago`;
    if (diffMins < 1440) return `${Math.floor(diffMins / 60)}h ago`;
    return new Date(date).toLocaleDateString();
  };

  const getActivityIcon = (type: Activity['type']) => {
    const icons = {
      login: '🔐',
      purchase: '🛒',
      comment: '💬',
      like: '❤️',
      share: '🔄'
    };
    return icons[type];
  };

  return (
    <div className="activity-item">
      <span className="activity-icon">{getActivityIcon(activity.type)}</span>
      <div className="activity-content">
        <p className="activity-description">{activity.description}</p>
        {activity.metadata?.productName && (
          <span className="activity-meta">
            Product: {activity.metadata.productName}
          </span>
        )}
      </div>
      <div className="activity-actions">
        <button 
          onClick={handleLike}
          className={isLiked ? 'liked' : ''}
        >
          {isLiked ? '❤️' : '🤍'} {likeCount}
        </button>
        <span className="activity-time">{formatTimestamp(activity.timestamp)}</span>
      </div>
    </div>
  );
}
```

Now let's add filtering to the feed:

```tsx
// src/components/ActivityFeed.tsx - Version 4 (With Filtering)
import { useState, useEffect } from 'react';
import { Activity } from '../types/activity';
import { ActivityItem } from './ActivityItem';

type ActivityFilter = 'all' | Activity['type'];

export function ActivityFeed({ userId }: { userId: string }) {
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [filter, setFilter] = useState<ActivityFilter>('all');

  useEffect(() => {
    fetch(`/api/users/${userId}/activities`)
      .then(res => res.json())
      .then(data => {
        setActivities(data);
        setIsLoading(false);
      });
  }, [userId]);

  const handleLike = (activityId: string) => {
    console.log('Liked activity:', activityId);
    // In a real app, this would make an API call
  };

  const filteredActivities = filter === 'all' 
    ? activities 
    : activities.filter(a => a.type === filter);

  if (isLoading) {
    return <div>Loading activities...</div>;
  }

  return (
    <div className="activity-feed">
      <div className="activity-header">
        <h2>Recent Activity</h2>
        <div className="activity-filters">
          <button 
            onClick={() => setFilter('all')}
            className={filter === 'all' ? 'active' : ''}
          >
            All
          </button>
          <button 
            onClick={() => setFilter('purchase')}
            className={filter === 'purchase' ? 'active' : ''}
          >
            Purchases
          </button>
          <button 
            onClick={() => setFilter('comment')}
            className={filter === 'comment' ? 'active' : ''}
          >
            Comments
          </button>
          <button 
            onClick={() => setFilter('like')}
            className={filter === 'like' ? 'active' : ''}
          >
            Likes
          </button>
        </div>
      </div>
      <div className="activity-list">
        {filteredActivities.map(activity => (
          <ActivityItem 
            key={activity.id} 
            activity={activity}
            onLike={handleLike}
          />
        ))}
      </div>
    </div>
  );
}
```

This works correctly. Now let's see what happens when we use the wrong key.

### The Failure: Using Array Index as Key

Let's intentionally break our code by using array index as the key:

```tsx
// src/components/ActivityFeed.tsx - BROKEN VERSION (Index as Key)
export function ActivityFeed({ userId }: { userId: string }) {
  // ... (same state and effects as before)

  return (
    <div className="activity-feed">
      <div className="activity-header">
        {/* ... (same filters as before) */}
      </div>
      <div className="activity-list">
        {filteredActivities.map((activity, index) => (
          <ActivityItem 
            key={index}  // ← WRONG: Using index as key
            activity={activity}
            onLike={handleLike}
          />
        ))}
      </div>
    </div>
  );
}
```

### Diagnostic Analysis: The Wrong Item Gets Updated

**Test scenario**:
1. Load the activity feed (shows 10 activities)
2. Like the 3rd activity (Purchase: "Bought Premium Plan")
3. Click "Purchases" filter (now shows only 3 purchase activities)
4. Observe which activity shows as liked

**Browser Behavior**:
The WRONG activity shows as liked. The 3rd item in the filtered list is liked, even though it's a different activity than the one we actually liked.

**React DevTools Evidence - Components Tab**:
Before filtering:
```
ActivityFeed
  ├─ ActivityItem (key: 0) - Login
  ├─ ActivityItem (key: 1) - Comment
  ├─ ActivityItem (key: 2) - Purchase ← We liked this one
  ├─ ActivityItem (key: 3) - Like
  └─ ...
```

After filtering to "Purchases":
```
ActivityFeed
  ├─ ActivityItem (key: 0) - Purchase (different activity)
  ├─ ActivityItem (key: 1) - Purchase (different activity)
  ├─ ActivityItem (key: 2) - Purchase ← Shows as liked (WRONG!)
  └─ ...
```

**What React DevTools shows**:
- The component at index 2 retained its state (`isLiked: true`)
- But it's now rendering a DIFFERENT activity
- React reused the component instance because the key (index 2) stayed the same

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Only the "Premium Plan" purchase should show as liked
   - Actual: A completely different purchase shows as liked

2. **What React DevTools reveals**:
   - The component at key `2` kept its internal state
   - But the `activity` prop changed to a different activity
   - React thought it was the same component (same key), so it preserved state

3. **Root cause identified**:
   When the array changes (filtering), the indices shift. React sees the same key (index 2) and thinks it's the same component, so it preserves the component's internal state (`isLiked`, `likeCount`). But the `activity` prop is now pointing to a different activity.

4. **Why index keys can't solve this**:
   Array indices are not stable identifiers. They change when the array is filtered, sorted, or reordered. React needs stable keys that move with the data.

5. **What we need**:
   Keys that are tied to the data itself, not its position in the array.

### The Fix: Using Stable Keys

Let's fix this by using the activity's unique ID:

```tsx
// src/components/ActivityFeed.tsx - FIXED VERSION (Proper Keys)
export function ActivityFeed({ userId }: { userId: string }) {
  // ... (same state and effects as before)

  return (
    <div className="activity-feed">
      <div className="activity-header">
        {/* ... (same filters as before) */}
      </div>
      <div className="activity-list">
        {filteredActivities.map(activity => (
          <ActivityItem 
            key={activity.id}  // ✓ CORRECT: Using stable ID
            activity={activity}
            onLike={handleLike}
          />
        ))}
      </div>
    </div>
  );
}
```

**Verification**:
1. Load the activity feed
2. Like the "Premium Plan" purchase
3. Filter to "Purchases"
4. Result: Only the "Premium Plan" purchase shows as liked ✓

**React DevTools Evidence - After Fix**:
Before filtering:
```
ActivityFeed
  ├─ ActivityItem (key: "act_123") - Login
  ├─ ActivityItem (key: "act_124") - Comment
  ├─ ActivityItem (key: "act_125") - Purchase ← We liked this one
  ├─ ActivityItem (key: "act_126") - Like
  └─ ...
```

After filtering to "Purchases":
```
ActivityFeed
  ├─ ActivityItem (key: "act_125") - Purchase ← Still liked ✓
  ├─ ActivityItem (key: "act_130") - Purchase
  ├─ ActivityItem (key: "act_135") - Purchase
  └─ ...
```

**What changed**:
- The component with key `"act_125"` moved positions but kept its state
- React correctly identified it as the same component
- Other purchase activities have different keys, so they have their own independent state

### Common Failure Modes and Their Signatures

#### Symptom: State appears on wrong items after filtering/sorting

**Browser behavior**:
User interacts with one item, filters the list, and a different item shows the interaction state.

**Console pattern**:
```
(No errors - this is a logic bug, not a runtime error)
```

**DevTools clues**:
- Component tree shows same keys before and after filter
- Props change but state doesn't reset
- Highlighted updates show components reusing instances

**Root cause**: Using array index or non-unique values as keys

**Solution**: Use stable, unique identifiers from your data

#### Symptom: Components don't update when array is reordered

**Browser behavior**:
User sorts the list, but items appear in wrong order or with wrong data.

**Console pattern**:
```
(No errors)
```

**DevTools clues**:
- Component order in tree doesn't match visual order
- Props update but components don't re-render
- Keys are duplicated or based on position

**Root cause**: Keys don't uniquely identify items

**Solution**: Ensure each key is unique and stable across renders

#### Symptom: Input fields lose focus or reset when typing

**Browser behavior**:
User types in an input field within a list item, and the input loses focus or resets after each keystroke.

**Console pattern**:
```
Warning: Each child in a list should have a unique "key" prop.
```

**DevTools clues**:
- Component unmounts and remounts on each keystroke
- New component instance created each render
- Keys are generated inside render (e.g., `Math.random()`)

**Root cause**: Keys change on every render

**Solution**: Generate keys outside of render, or use stable data properties

### When to Apply This Solution

**What it optimizes for**:
- Correct component state preservation
- Efficient DOM updates
- Predictable behavior when lists change

**What it requires**:
- Unique identifiers in your data
- Understanding of your data's identity

**When to use stable keys**:
- Always, for any list that can change
- Especially when list items have internal state
- When lists can be filtered, sorted, or reordered
- When items can be added/removed

**When index keys might be acceptable** (rare):
- Static lists that never change
- Server-rendered lists with no client-side interaction
- Lists where items have no internal state and never reorder

**Code characteristics**:
- Setup: Minimal (just use existing ID field)
- Maintenance: None (keys are part of data structure)
- Performance: Optimal (React can efficiently reconcile changes)

## Conditional rendering patterns

## Conditional Rendering Patterns

Our activity feed now handles lists correctly. But what about showing different UI based on conditions? Empty states, loading states, error states—these are all forms of conditional rendering.

### Iteration 2: Adding Empty and Error States

Let's enhance our activity feed to handle edge cases:

```tsx
// src/components/ActivityFeed.tsx - Version 5 (With All States)
import { useState, useEffect } from 'react';
import { Activity } from '../types/activity';
import { ActivityItem } from './ActivityItem';

type ActivityFilter = 'all' | Activity['type'];

export function ActivityFeed({ userId }: { userId: string }) {
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [filter, setFilter] = useState<ActivityFilter>('all');

  useEffect(() => {
    setIsLoading(true);
    setError(null);
    
    fetch(`/api/users/${userId}/activities`)
      .then(res => {
        if (!res.ok) {
          throw new Error(`HTTP ${res.status}: ${res.statusText}`);
        }
        return res.json();
      })
      .then(data => {
        setActivities(data);
        setIsLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setIsLoading(false);
      });
  }, [userId]);

  const handleLike = (activityId: string) => {
    console.log('Liked activity:', activityId);
  };

  const filteredActivities = filter === 'all' 
    ? activities 
    : activities.filter(a => a.type === filter);

  // Loading state
  if (isLoading) {
    return (
      <div className="activity-feed">
        <div className="activity-loading">
          <div className="spinner" />
          <p>Loading activities...</p>
        </div>
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <div className="activity-feed">
        <div className="activity-error">
          <span className="error-icon">⚠️</span>
          <h3>Failed to load activities</h3>
          <p>{error}</p>
          <button onClick={() => window.location.reload()}>
            Try Again
          </button>
        </div>
      </div>
    );
  }

  // Empty state (no activities at all)
  if (activities.length === 0) {
    return (
      <div className="activity-feed">
        <div className="activity-empty">
          <span className="empty-icon">📭</span>
          <h3>No activity yet</h3>
          <p>Your recent actions will appear here.</p>
        </div>
      </div>
    );
  }

  return (
    <div className="activity-feed">
      <div className="activity-header">
        <h2>Recent Activity</h2>
        <div className="activity-filters">
          <button 
            onClick={() => setFilter('all')}
            className={filter === 'all' ? 'active' : ''}
          >
            All ({activities.length})
          </button>
          <button 
            onClick={() => setFilter('purchase')}
            className={filter === 'purchase' ? 'active' : ''}
          >
            Purchases ({activities.filter(a => a.type === 'purchase').length})
          </button>
          <button 
            onClick={() => setFilter('comment')}
            className={filter === 'comment' ? 'active' : ''}
          >
            Comments ({activities.filter(a => a.type === 'comment').length})
          </button>
          <button 
            onClick={() => setFilter('like')}
            className={filter === 'like' ? 'active' : ''}
          >
            Likes ({activities.filter(a => a.type === 'like').length})
          </button>
        </div>
      </div>
      
      {/* Empty state for filtered results */}
      {filteredActivities.length === 0 ? (
        <div className="activity-empty-filter">
          <p>No {filter} activities found.</p>
          <button onClick={() => setFilter('all')}>
            Show all activities
          </button>
        </div>
      ) : (
        <div className="activity-list">
          {filteredActivities.map(activity => (
            <ActivityItem 
              key={activity.id} 
              activity={activity}
              onLike={handleLike}
            />
          ))}
        </div>
      )}
    </div>
  );
}
```

This component now handles five distinct states:
1. **Loading**: Fetching data
2. **Error**: Fetch failed
3. **Empty**: No activities exist
4. **Filtered empty**: Activities exist, but none match the filter
5. **Success**: Activities to display

### Conditional Rendering Patterns

React offers several ways to conditionally render content. Let's examine each pattern and when to use it.

#### Pattern 1: Early Return (Guard Clauses)

**Best for**: Mutually exclusive states (loading, error, empty)

```tsx
function Component() {
  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;
  if (data.length === 0) return <EmptyState />;
  
  return <MainContent data={data} />;
}
```

**Advantages**:
- Clear, linear flow
- Easy to read and reason about
- No nesting

**When to use**:
- States are mutually exclusive
- Each state needs completely different UI
- You want to avoid deeply nested JSX

#### Pattern 2: Ternary Operator

**Best for**: Simple binary conditions

```tsx
function Component({ isPublic }: { isPublic: boolean }) {
  return (
    <div>
      {isPublic ? (
        <PublicContent />
      ) : (
        <PrivateContent />
      )}
    </div>
  );
}
```

**Advantages**:
- Concise for simple conditions
- Works inline in JSX

**When to use**:
- Two mutually exclusive options
- Both branches are simple
- Condition is straightforward

**Avoid when**:
- Nesting multiple ternaries (becomes unreadable)
- Either branch is complex (use early return instead)

#### Pattern 3: Logical AND (&&)

**Best for**: Conditionally showing a single element

```tsx
function Component({ showBanner, message }: { showBanner: boolean; message?: string }) {
  return (
    <div>
      <h1>Welcome</h1>
      {showBanner && <Banner />}
      {message && <Alert message={message} />}
    </div>
  );
}
```

**Advantages**:
- Very concise
- Clear intent: "show this if condition is true"

**When to use**:
- Showing/hiding a single element
- No "else" branch needed
- Condition is boolean or truthy/falsy

**Gotcha**: Be careful with numbers!

```tsx
// ❌ WRONG: Renders "0" when count is 0
{count && <p>You have {count} items</p>}

// ✓ CORRECT: Explicitly check for > 0
{count > 0 && <p>You have {count} items</p>}

// ✓ ALSO CORRECT: Convert to boolean
{!!count && <p>You have {count} items</p>}
```

#### Pattern 4: Nullish Coalescing for Defaults

**Best for**: Providing fallback content

```tsx
function UserProfile({ user }: { user?: User }) {
  return (
    <div>
      <h1>{user?.name ?? 'Anonymous User'}</h1>
      <p>{user?.bio ?? 'No bio provided'}</p>
      <img src={user?.avatar ?? '/default-avatar.png'} alt="Avatar" />
    </div>
  );
}
```

**Advantages**:
- Concise fallback values
- Handles `null` and `undefined` (but not `0` or `''`)

**When to use**:
- Providing default values
- Handling optional data
- You want to preserve falsy values like `0` or `''`

#### Pattern 5: Switch Statements (via Object Mapping)

**Best for**: Multiple distinct states with different UI

```tsx
type Status = 'idle' | 'loading' | 'success' | 'error';

function Component({ status }: { status: Status }) {
  const statusComponents = {
    idle: <IdleState />,
    loading: <LoadingSpinner />,
    success: <SuccessMessage />,
    error: <ErrorMessage />
  };

  return (
    <div>
      {statusComponents[status]}
    </div>
  );
}
```

**Advantages**:
- Scales well with many states
- Easy to add new states
- TypeScript ensures all states are handled

**When to use**:
- 3+ distinct states
- Each state has different UI
- States are represented by a union type

#### Pattern 6: Render Props / Children Functions

**Best for**: Delegating rendering logic to parent

```tsx
interface DataFetcherProps<T> {
  url: string;
  children: (data: T | null, isLoading: boolean, error: Error | null) => React.ReactNode;
}

function DataFetcher<T>({ url, children }: DataFetcherProps<T>) {
  const [data, setData] = useState<T | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data);
        setIsLoading(false);
      })
      .catch(err => {
        setError(err);
        setIsLoading(false);
      });
  }, [url]);

  return <>{children(data, isLoading, error)}</>;
}

// Usage
function ActivityFeedWrapper() {
  return (
    <DataFetcher<Activity[]> url="/api/activities">
      {(activities, isLoading, error) => {
        if (isLoading) return <LoadingSpinner />;
        if (error) return <ErrorMessage error={error} />;
        if (!activities || activities.length === 0) return <EmptyState />;
        return <ActivityList activities={activities} />;
      }}
    </DataFetcher>
  );
}
```

**Advantages**:
- Separates data fetching from rendering
- Reusable data fetching logic
- Parent controls rendering

**When to use**:
- Building reusable data fetching components
- Complex conditional logic that varies by use case
- You want to separate concerns

### Iteration 3: Extracting State Components

As our conditional rendering grows, let's extract each state into its own component:

```tsx
// src/components/ActivityFeed/LoadingState.tsx
export function LoadingState() {
  return (
    <div className="activity-loading">
      <div className="spinner" />
      <p>Loading activities...</p>
    </div>
  );
}
```

```tsx
// src/components/ActivityFeed/ErrorState.tsx
interface ErrorStateProps {
  error: string;
  onRetry: () => void;
}

export function ErrorState({ error, onRetry }: ErrorStateProps) {
  return (
    <div className="activity-error">
      <span className="error-icon">⚠️</span>
      <h3>Failed to load activities</h3>
      <p>{error}</p>
      <button onClick={onRetry}>Try Again</button>
    </div>
  );
}
```

```tsx
// src/components/ActivityFeed/EmptyState.tsx
interface EmptyStateProps {
  isFiltered?: boolean;
  filterType?: string;
  onClearFilter?: () => void;
}

export function EmptyState({ isFiltered, filterType, onClearFilter }: EmptyStateProps) {
  if (isFiltered) {
    return (
      <div className="activity-empty-filter">
        <p>No {filterType} activities found.</p>
        <button onClick={onClearFilter}>Show all activities</button>
      </div>
    );
  }

  return (
    <div className="activity-empty">
      <span className="empty-icon">📭</span>
      <h3>No activity yet</h3>
      <p>Your recent actions will appear here.</p>
    </div>
  );
}
```

```tsx
// src/components/ActivityFeed.tsx - Version 6 (Refactored)
import { useState, useEffect } from 'react';
import { Activity } from '../types/activity';
import { ActivityItem } from './ActivityItem';
import { LoadingState } from './ActivityFeed/LoadingState';
import { ErrorState } from './ActivityFeed/ErrorState';
import { EmptyState } from './ActivityFeed/EmptyState';

type ActivityFilter = 'all' | Activity['type'];

export function ActivityFeed({ userId }: { userId: string }) {
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [filter, setFilter] = useState<ActivityFilter>('all');

  const fetchActivities = () => {
    setIsLoading(true);
    setError(null);
    
    fetch(`/api/users/${userId}/activities`)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => {
        setActivities(data);
        setIsLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setIsLoading(false);
      });
  };

  useEffect(() => {
    fetchActivities();
  }, [userId]);

  const handleLike = (activityId: string) => {
    console.log('Liked activity:', activityId);
  };

  const filteredActivities = filter === 'all' 
    ? activities 
    : activities.filter(a => a.type === filter);

  // Early returns for distinct states
  if (isLoading) {
    return (
      <div className="activity-feed">
        <LoadingState />
      </div>
    );
  }

  if (error) {
    return (
      <div className="activity-feed">
        <ErrorState error={error} onRetry={fetchActivities} />
      </div>
    );
  }

  if (activities.length === 0) {
    return (
      <div className="activity-feed">
        <EmptyState />
      </div>
    );
  }

  return (
    <div className="activity-feed">
      <div className="activity-header">
        <h2>Recent Activity</h2>
        <div className="activity-filters">
          <button 
            onClick={() => setFilter('all')}
            className={filter === 'all' ? 'active' : ''}
          >
            All ({activities.length})
          </button>
          <button 
            onClick={() => setFilter('purchase')}
            className={filter === 'purchase' ? 'active' : ''}
          >
            Purchases ({activities.filter(a => a.type === 'purchase').length})
          </button>
          <button 
            onClick={() => setFilter('comment')}
            className={filter === 'comment' ? 'active' : ''}
          >
            Comments ({activities.filter(a => a.type === 'comment').length})
          </button>
        </div>
      </div>
      
      {filteredActivities.length === 0 ? (
        <EmptyState 
          isFiltered 
          filterType={filter}
          onClearFilter={() => setFilter('all')}
        />
      ) : (
        <div className="activity-list">
          {filteredActivities.map(activity => (
            <ActivityItem 
              key={activity.id} 
              activity={activity}
              onLike={handleLike}
            />
          ))}
        </div>
      )}
    </div>
  );
}
```

**Improvements**:
- Each state has its own component
- Main component is easier to read
- State components are reusable and testable
- Clear separation of concerns

### Common Failure Modes and Their Signatures

#### Symptom: Renders "0" or "false" in the UI

**Browser behavior**:
Instead of showing nothing, the number `0` or the word `false` appears in the UI.

**Console pattern**:
```
(No errors - this is expected React behavior)
```

**Example**:

```tsx
// ❌ Renders "0" when count is 0
{count && <p>Items: {count}</p>}

// ✓ Fixed
{count > 0 && <p>Items: {count}</p>}
```

**Root cause**: `&&` operator returns the left side if falsy. React renders `0` and `false` as text.

**Solution**: Use explicit boolean conditions

#### Symptom: Ternary becomes unreadable

**Browser behavior**:
Code works but is impossible to maintain.

**Example**:

```tsx
// ❌ Nested ternary hell
{isLoading ? (
  <Spinner />
) : error ? (
  <Error />
) : data.length === 0 ? (
  <Empty />
) : (
  <List data={data} />
)}

// ✓ Use early returns instead
if (isLoading) return <Spinner />;
if (error) return <Error />;
if (data.length === 0) return <Empty />;
return <List data={data} />;
```

**Root cause**: Ternaries don't scale beyond 2 branches

**Solution**: Use early returns or object mapping for multiple states

#### Symptom: Condition always evaluates to true/false

**Browser behavior**:
Component always shows or never shows, regardless of actual state.

**Example**:

```tsx
// ❌ String "false" is truthy
{user.isAdmin === "false" && <AdminPanel />}

// ✓ Compare to boolean
{user.isAdmin === false && <AdminPanel />}

// ✓ Or use strict equality
{!user.isAdmin && <AdminPanel />}
```

**Root cause**: Type coercion or incorrect comparison

**Solution**: Use TypeScript and strict equality checks

### When to Apply These Patterns

**Early Returns**:
- Use for: Mutually exclusive states (loading, error, success)
- Optimizes for: Readability, linear flow
- Avoid when: States can overlap or you need to show multiple things

**Ternary Operator**:
- Use for: Simple binary conditions
- Optimizes for: Conciseness
- Avoid when: Nesting more than one level

**Logical AND (&&)**:
- Use for: Showing/hiding single elements
- Optimizes for: Brevity
- Avoid when: Left side can be `0`, `""`, or other falsy values you want to render

**Object Mapping**:
- Use for: 3+ distinct states
- Optimizes for: Scalability, type safety
- Avoid when: States are not mutually exclusive

**Component Extraction**:
- Use for: Complex conditional UI
- Optimizes for: Reusability, testability
- Avoid when: Component is used only once and is simple

## Avoiding unnecessary re-renders

## Avoiding Unnecessary Re-Renders

Our activity feed now handles lists, keys, and conditional rendering correctly. But there's one more problem: **performance**. What happens when we have hundreds of activities and the list re-renders frequently?

### The Failure: Entire List Re-renders on One Item Change

Let's add a feature that updates activity timestamps in real-time (e.g., "2m ago" → "3m ago"). This will expose a performance problem.

```tsx
// src/components/ActivityFeed.tsx - Version 7 (With Real-time Updates)
import { useState, useEffect } from 'react';
import { Activity } from '../types/activity';
import { ActivityItem } from './ActivityItem';
import { LoadingState } from './ActivityFeed/LoadingState';
import { ErrorState } from './ActivityFeed/ErrorState';
import { EmptyState } from './ActivityFeed/EmptyState';

type ActivityFilter = 'all' | Activity['type'];

export function ActivityFeed({ userId }: { userId: string }) {
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [filter, setFilter] = useState<ActivityFilter>('all');
  const [currentTime, setCurrentTime] = useState(new Date());

  // Update current time every minute to refresh "X minutes ago" displays
  useEffect(() => {
    const interval = setInterval(() => {
      setCurrentTime(new Date());
    }, 60000); // Every 60 seconds

    return () => clearInterval(interval);
  }, []);

  const fetchActivities = () => {
    setIsLoading(true);
    setError(null);
    
    fetch(`/api/users/${userId}/activities`)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => {
        setActivities(data);
        setIsLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setIsLoading(false);
      });
  };

  useEffect(() => {
    fetchActivities();
  }, [userId]);

  const handleLike = (activityId: string) => {
    console.log('Liked activity:', activityId);
  };

  const filteredActivities = filter === 'all' 
    ? activities 
    : activities.filter(a => a.type === filter);

  if (isLoading) return <div className="activity-feed"><LoadingState /></div>;
  if (error) return <div className="activity-feed"><ErrorState error={error} onRetry={fetchActivities} /></div>;
  if (activities.length === 0) return <div className="activity-feed"><EmptyState /></div>;

  return (
    <div className="activity-feed">
      <div className="activity-header">
        <h2>Recent Activity</h2>
        <div className="activity-filters">
          <button 
            onClick={() => setFilter('all')}
            className={filter === 'all' ? 'active' : ''}
          >
            All ({activities.length})
          </button>
          <button 
            onClick={() => setFilter('purchase')}
            className={filter === 'purchase' ? 'active' : ''}
          >
            Purchases ({activities.filter(a => a.type === 'purchase').length})
          </button>
        </div>
      </div>
      
      {filteredActivities.length === 0 ? (
        <EmptyState 
          isFiltered 
          filterType={filter}
          onClearFilter={() => setFilter('all')}
        />
      ) : (
        <div className="activity-list">
          {filteredActivities.map(activity => (
            <ActivityItem 
              key={activity.id} 
              activity={activity}
              currentTime={currentTime}  // ← Pass current time to each item
              onLike={handleLike}
            />
          ))}
        </div>
      )}
    </div>
  );
}
```

```tsx
// src/components/ActivityItem.tsx - Version 3 (With Current Time Prop)
import { useState } from 'react';
import { Activity } from '../types/activity';

interface ActivityItemProps {
  activity: Activity;
  currentTime: Date;  // ← New prop
  onLike: (activityId: string) => void;
}

export function ActivityItem({ activity, currentTime, onLike }: ActivityItemProps) {
  const [isLiked, setIsLiked] = useState(false);
  const [likeCount, setLikeCount] = useState(activity.metadata?.likes || 0);

  // Add console.log to track renders
  console.log(`Rendering ActivityItem: ${activity.id}`);

  const handleLike = () => {
    setIsLiked(!isLiked);
    setLikeCount(prev => isLiked ? prev - 1 : prev + 1);
    onLike(activity.id);
  };

  const formatTimestamp = (date: Date, now: Date) => {
    const diffMs = now.getTime() - new Date(date).getTime();
    const diffMins = Math.floor(diffMs / 60000);
    
    if (diffMins < 1) return 'Just now';
    if (diffMins < 60) return `${diffMins}m ago`;
    if (diffMins < 1440) return `${Math.floor(diffMins / 60)}h ago`;
    return new Date(date).toLocaleDateString();
  };

  const getActivityIcon = (type: Activity['type']) => {
    const icons = {
      login: '🔐',
      purchase: '🛒',
      comment: '💬',
      like: '❤️',
      share: '🔄'
    };
    return icons[type];
  };

  return (
    <div className="activity-item">
      <span className="activity-icon">{getActivityIcon(activity.type)}</span>
      <div className="activity-content">
        <p className="activity-description">{activity.description}</p>
        {activity.metadata?.productName && (
          <span className="activity-meta">
            Product: {activity.metadata.productName}
          </span>
        )}
      </div>
      <div className="activity-actions">
        <button 
          onClick={handleLike}
          className={isLiked ? 'liked' : ''}
        >
          {isLiked ? '❤️' : '🤍'} {likeCount}
        </button>
        <span className="activity-time">
          {formatTimestamp(activity.timestamp, currentTime)}
        </span>
      </div>
    </div>
  );
}
```

### Diagnostic Analysis: Reading the Performance Problem

**Test scenario**:
1. Load activity feed with 50 activities
2. Wait 60 seconds for the timer to tick
3. Observe browser console

**Browser Console**:
```
Rendering ActivityItem: act_001
Rendering ActivityItem: act_002
Rendering ActivityItem: act_003
... (50 times)
Rendering ActivityItem: act_050
```

Every 60 seconds, all 50 console.log statements fire. Every single `ActivityItem` re-renders, even though only the timestamps changed.

**React DevTools Evidence - Profiler Tab**:
1. Open React DevTools
2. Go to Profiler tab
3. Click "Record"
4. Wait for timer to tick (or manually change `currentTime` state)
5. Stop recording

**Profiler shows**:
- `ActivityFeed` rendered: 1 time (2.3ms)
- `ActivityItem` rendered: 50 times (total: 45ms)
- Each `ActivityItem` took ~0.9ms
- Reason for each render: "Props changed"

**React DevTools Evidence - Components Tab with Highlight Updates**:
1. Open React DevTools
2. Go to Components tab
3. Click the "⚙️" icon → Enable "Highlight updates when components render"
4. Wait for timer to tick

**What you see**:
Every single `ActivityItem` flashes with a colored border, indicating they all re-rendered.

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Smooth, imperceptible updates
   - Actual: With 50 items, no visible lag yet. But with 500 items, the UI would stutter.

2. **What the console reveals**:
   - All 50 items log on every timer tick
   - This means all 50 components are executing their render function

3. **What React DevTools shows**:
   - Profiler: 50 separate render operations
   - Highlight updates: All items flash simultaneously
   - Props changed: `currentTime` prop is new on every tick

4. **Root cause identified**:
   When `currentTime` state updates in `ActivityFeed`, React re-renders the component. This creates a new `currentTime` value that gets passed to every `ActivityItem`. React sees the prop changed and re-renders all items.

5. **Why the current approach can't scale**:
   - With 500 activities, that's 500 re-renders every 60 seconds
   - Each render executes the entire component function
   - Even though most items' timestamps don't actually change (e.g., "2 hours ago" stays "2 hours ago")

6. **What we need**:
   A way to tell React: "Only re-render this component if its output would actually be different."

### The Concept: React.memo and Memoization

**Memoization** is a technique where you cache the result of an expensive computation and reuse it if the inputs haven't changed.

React provides `React.memo()` to memoize components. It works like this:

1. React renders your component with props `{ a: 1, b: 2 }`
2. React.memo caches the output
3. Next render, props are `{ a: 1, b: 2 }` (same values)
4. React.memo returns the cached output without re-rendering
5. Next render, props are `{ a: 1, b: 3 }` (b changed)
6. React.memo re-renders because props changed

**The key insight**: React.memo does a **shallow comparison** of props. If all props are the same (using `Object.is` comparison), React skips the render.

### Iteration 1: Applying React.memo

Let's wrap `ActivityItem` with `React.memo`:

```tsx
// src/components/ActivityItem.tsx - Version 4 (With React.memo)
import { useState, memo } from 'react';
import { Activity } from '../types/activity';

interface ActivityItemProps {
  activity: Activity;
  currentTime: Date;
  onLike: (activityId: string) => void;
}

// Wrap the component with memo
export const ActivityItem = memo(function ActivityItem({ 
  activity, 
  currentTime, 
  onLike 
}: ActivityItemProps) {
  const [isLiked, setIsLiked] = useState(false);
  const [likeCount, setLikeCount] = useState(activity.metadata?.likes || 0);

  console.log(`Rendering ActivityItem: ${activity.id}`);

  const handleLike = () => {
    setIsLiked(!isLiked);
    setLikeCount(prev => isLiked ? prev - 1 : prev + 1);
    onLike(activity.id);
  };

  const formatTimestamp = (date: Date, now: Date) => {
    const diffMs = now.getTime() - new Date(date).getTime();
    const diffMins = Math.floor(diffMs / 60000);
    
    if (diffMins < 1) return 'Just now';
    if (diffMins < 60) return `${diffMins}m ago`;
    if (diffMins < 1440) return `${Math.floor(diffMins / 60)}h ago`;
    return new Date(date).toLocaleDateString();
  };

  const getActivityIcon = (type: Activity['type']) => {
    const icons = {
      login: '🔐',
      purchase: '🛒',
      comment: '💬',
      like: '❤️',
      share: '🔄'
    };
    return icons[type];
  };

  return (
    <div className="activity-item">
      <span className="activity-icon">{getActivityIcon(activity.type)}</span>
      <div className="activity-content">
        <p className="activity-description">{activity.description}</p>
        {activity.metadata?.productName && (
          <span className="activity-meta">
            Product: {activity.metadata.productName}
          </span>
        )}
      </div>
      <div className="activity-actions">
        <button 
          onClick={handleLike}
          className={isLiked ? 'liked' : ''}
        >
          {isLiked ? '❤️' : '🤍'} {likeCount}
        </button>
        <span className="activity-time">
          {formatTimestamp(activity.timestamp, currentTime)}
        </span>
      </div>
    </div>
  );
});
```

**Verification**:
Wait 60 seconds for the timer to tick...

**Browser Console**:
```
Rendering ActivityItem: act_001
Rendering ActivityItem: act_002
... (still 50 times!)
```

**Wait, what?** We added `React.memo` but all items still re-render!

### Diagnostic Analysis: Why React.memo Didn't Help

**React DevTools Evidence - Profiler**:
- All 50 `ActivityItem` components still rendered
- Reason: "Props changed"

**Let's investigate which prop changed**:

```tsx
// Add this inside ActivityItem to debug
console.log('Props:', {
  activityId: activity.id,
  currentTime: currentTime.toISOString(),
  onLike: onLike.toString()
});
```

**Console output on first render**:
```
Props: {
  activityId: "act_001",
  currentTime: "2024-01-15T10:30:00.000Z",
  onLike: "function onLike() { ... }"
}
```

**Console output on second render (after timer tick)**:
```
Props: {
  activityId: "act_001",
  currentTime: "2024-01-15T10:31:00.000Z",  ← Changed!
  onLike: "function onLike() { ... }"
}
```

**Root cause identified**:
The `currentTime` prop is a new `Date` object on every render. Even though the time value changed by only 60 seconds, it's a completely new object reference. React.memo's shallow comparison sees:

```javascript
oldProps.currentTime === newProps.currentTime  // false (different objects)
```

So React.memo thinks the props changed and re-renders the component.

**But there's another problem**: The `onLike` function is also recreated on every render of `ActivityFeed`. Let's verify:

```tsx
// In ActivityFeed, add this:
const handleLike = (activityId: string) => {
  console.log('Liked activity:', activityId);
};

// This function is recreated on every render of ActivityFeed
// So every ActivityItem gets a new function reference
```

### The Failure: Reference Equality vs. Value Equality

This is a fundamental JavaScript concept that trips up many React developers.

**Primitive values** (numbers, strings, booleans) are compared by value:

```javascript
const a = 5;
const b = 5;
console.log(a === b);  // true

const x = "hello";
const y = "hello";
console.log(x === y);  // true
```

**Objects, arrays, and functions** are compared by reference:

```javascript
const obj1 = { name: "Alice" };
const obj2 = { name: "Alice" };
console.log(obj1 === obj2);  // false (different objects)

const arr1 = [1, 2, 3];
const arr2 = [1, 2, 3];
console.log(arr1 === arr2);  // false (different arrays)

const fn1 = () => console.log("hi");
const fn2 = () => console.log("hi");
console.log(fn1 === fn2);  // false (different functions)
```

**In React**:
Every time a component re-renders, any objects, arrays, or functions defined inside it are recreated with new references. React.memo sees these as "changed" even if their content is identical.

### Solution: useCallback for Function Stability

React provides `useCallback` to memoize functions:

```tsx
// src/components/ActivityFeed.tsx - Version 8 (With useCallback)
import { useState, useEffect, useCallback } from 'react';
import { Activity } from '../types/activity';
import { ActivityItem } from './ActivityItem';
import { LoadingState } from './ActivityFeed/LoadingState';
import { ErrorState } from './ActivityFeed/ErrorState';
import { EmptyState } from './ActivityFeed/EmptyState';

type ActivityFilter = 'all' | Activity['type'];

export function ActivityFeed({ userId }: { userId: string }) {
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [filter, setFilter] = useState<ActivityFilter>('all');
  const [currentTime, setCurrentTime] = useState(new Date());

  useEffect(() => {
    const interval = setInterval(() => {
      setCurrentTime(new Date());
    }, 60000);
    return () => clearInterval(interval);
  }, []);

  const fetchActivities = useCallback(() => {
    setIsLoading(true);
    setError(null);
    
    fetch(`/api/users/${userId}/activities`)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => {
        setActivities(data);
        setIsLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setIsLoading(false);
      });
  }, [userId]);  // Only recreate if userId changes

  useEffect(() => {
    fetchActivities();
  }, [fetchActivities]);

  // Memoize the like handler
  const handleLike = useCallback((activityId: string) => {
    console.log('Liked activity:', activityId);
    // In a real app, this would make an API call
  }, []);  // No dependencies, so function never changes

  const filteredActivities = filter === 'all' 
    ? activities 
    : activities.filter(a => a.type === filter);

  if (isLoading) return <div className="activity-feed"><LoadingState /></div>;
  if (error) return <div className="activity-feed"><ErrorState error={error} onRetry={fetchActivities} /></div>;
  if (activities.length === 0) return <div className="activity-feed"><EmptyState /></div>;

  return (
    <div className="activity-feed">
      <div className="activity-header">
        <h2>Recent Activity</h2>
        <div className="activity-filters">
          <button 
            onClick={() => setFilter('all')}
            className={filter === 'all' ? 'active' : ''}
          >
            All ({activities.length})
          </button>
          <button 
            onClick={() => setFilter('purchase')}
            className={filter === 'purchase' ? 'active' : ''}
          >
            Purchases ({activities.filter(a => a.type === 'purchase').length})
          </button>
        </div>
      </div>
      
      {filteredActivities.length === 0 ? (
        <EmptyState 
          isFiltered 
          filterType={filter}
          onClearFilter={() => setFilter('all')}
        />
      ) : (
        <div className="activity-list">
          {filteredActivities.map(activity => (
            <ActivityItem 
              key={activity.id} 
              activity={activity}
              currentTime={currentTime}
              onLike={handleLike}  // ← Now stable across renders
            />
          ))}
        </div>
      )}
    </div>
  );
}
```

**What changed**:
- Wrapped `handleLike` with `useCallback`
- Added empty dependency array `[]` (function never changes)
- Wrapped `fetchActivities` with `useCallback` too (depends on `userId`)

**Verification**:
Wait 60 seconds for the timer to tick...

**Browser Console**:
```
Rendering ActivityItem: act_001
Rendering ActivityItem: act_002
... (still 50 times!)
```

**Still not working!** The `onLike` function is now stable, but `currentTime` is still a new object on every render.

### The Real Problem: Passing Unnecessary Props

Here's the key insight: **Do we actually need to pass `currentTime` as a prop?**

Each `ActivityItem` only needs to know its own `activity.timestamp`. It can calculate the relative time itself using the current time. We don't need to pass `currentTime` from the parent.

### Iteration 2: Removing Unnecessary Props

Let's refactor to remove the `currentTime` prop entirely:

```tsx
// src/components/ActivityItem.tsx - Version 5 (Self-contained Time Formatting)
import { useState, useEffect, memo } from 'react';
import { Activity } from '../types/activity';

interface ActivityItemProps {
  activity: Activity;
  onLike: (activityId: string) => void;
}

export const ActivityItem = memo(function ActivityItem({ 
  activity, 
  onLike 
}: ActivityItemProps) {
  const [isLiked, setIsLiked] = useState(false);
  const [likeCount, setLikeCount] = useState(activity.metadata?.likes || 0);
  const [relativeTime, setRelativeTime] = useState('');

  console.log(`Rendering ActivityItem: ${activity.id}`);

  // Each item manages its own time updates
  useEffect(() => {
    const updateTime = () => {
      const now = new Date();
      const diffMs = now.getTime() - new Date(activity.timestamp).getTime();
      const diffMins = Math.floor(diffMs / 60000);
      
      if (diffMins < 1) setRelativeTime('Just now');
      else if (diffMins < 60) setRelativeTime(`${diffMins}m ago`);
      else if (diffMins < 1440) setRelativeTime(`${Math.floor(diffMins / 60)}h ago`);
      else setRelativeTime(new Date(activity.timestamp).toLocaleDateString());
    };

    updateTime();  // Initial update
    const interval = setInterval(updateTime, 60000);  // Update every minute
    return () => clearInterval(interval);
  }, [activity.timestamp]);

  const handleLike = () => {
    setIsLiked(!isLiked);
    setLikeCount(prev => isLiked ? prev - 1 : prev + 1);
    onLike(activity.id);
  };

  const getActivityIcon = (type: Activity['type']) => {
    const icons = {
      login: '🔐',
      purchase: '🛒',
      comment: '💬',
      like: '❤️',
      share: '🔄'
    };
    return icons[type];
  };

  return (
    <div className="activity-item">
      <span className="activity-icon">{getActivityIcon(activity.type)}</span>
      <div className="activity-content">
        <p className="activity-description">{activity.description}</p>
        {activity.metadata?.productName && (
          <span className="activity-meta">
            Product: {activity.metadata.productName}
          </span>
        )}
      </div>
      <div className="activity-actions">
        <button 
          onClick={handleLike}
          className={isLiked ? 'liked' : ''}
        >
          {isLiked ? '❤️' : '🤍'} {likeCount}
        </button>
        <span className="activity-time">{relativeTime}</span>
      </div>
    </div>
  );
});
```

```tsx
// src/components/ActivityFeed.tsx - Version 9 (No currentTime Prop)
import { useState, useEffect, useCallback } from 'react';
import { Activity } from '../types/activity';
import { ActivityItem } from './ActivityItem';
import { LoadingState } from './ActivityFeed/LoadingState';
import { ErrorState } from './ActivityFeed/ErrorState';
import { EmptyState } from './ActivityFeed/EmptyState';

type ActivityFilter = 'all' | Activity['type'];

export function ActivityFeed({ userId }: { userId: string }) {
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [filter, setFilter] = useState<ActivityFilter>('all');

  const fetchActivities = useCallback(() => {
    setIsLoading(true);
    setError(null);
    
    fetch(`/api/users/${userId}/activities`)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => {
        setActivities(data);
        setIsLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setIsLoading(false);
      });
  }, [userId]);

  useEffect(() => {
    fetchActivities();
  }, [fetchActivities]);

  const handleLike = useCallback((activityId: string) => {
    console.log('Liked activity:', activityId);
  }, []);

  const filteredActivities = filter === 'all' 
    ? activities 
    : activities.filter(a => a.type === filter);

  if (isLoading) return <div className="activity-feed"><LoadingState /></div>;
  if (error) return <div className="activity-feed"><ErrorState error={error} onRetry={fetchActivities} /></div>;
  if (activities.length === 0) return <div className="activity-feed"><EmptyState /></div>;

  return (
    <div className="activity-feed">
      <div className="activity-header">
        <h2>Recent Activity</h2>
        <div className="activity-filters">
          <button 
            onClick={() => setFilter('all')}
            className={filter === 'all' ? 'active' : ''}
          >
            All ({activities.length})
          </button>
          <button 
            onClick={() => setFilter('purchase')}
            className={filter === 'purchase' ? 'active' : ''}
          >
            Purchases ({activities.filter(a => a.type === 'purchase').length})
          </button>
        </div>
      </div>
      
      {filteredActivities.length === 0 ? (
        <EmptyState 
          isFiltered 
          filterType={filter}
          onClearFilter={() => setFilter('all')}
        />
      ) : (
        <div className="activity-list">
          {filteredActivities.map(activity => (
            <ActivityItem 
              key={activity.id} 
              activity={activity}
              onLike={handleLike}
            />
          ))}
        </div>
      )}
    </div>
  );
}
```

**What changed**:
- Removed `currentTime` state from `ActivityFeed`
- Removed `currentTime` prop from `ActivityItem`
- Each `ActivityItem` now manages its own timer with `useEffect`
- Each item updates independently

**Verification**:
Wait 60 seconds...

**Browser Console**:
```
Rendering ActivityItem: act_001
```

Only ONE item re-renders! The item whose timestamp just crossed a minute boundary (e.g., from "2m ago" to "3m ago").

**React DevTools Evidence - Profiler**:
- `ActivityFeed` rendered: 0 times (not triggered)
- `ActivityItem` rendered: 1 time (only the one that needed to update)
- Total render time: 0.8ms (vs. 45ms before)

**React DevTools Evidence - Highlight Updates**:
Only one item flashes when its timer updates. The other 49 items remain static.

**Performance improvement**:
- Before: 50 re-renders every 60 seconds = 45ms
- After: 1 re-render every 60 seconds = 0.8ms
- **56x faster** ✓

### The Lesson: Optimize Component Boundaries

The best optimization is often **not using optimization techniques at all**, but rather **designing better component boundaries**.

**Before**: Parent managed time, passed it to all children
- Every child re-rendered when parent's time changed
- Needed React.memo and useCallback to prevent re-renders

**After**: Each child manages its own time
- Only the child that needs to update re-renders
- No React.memo or useCallback needed for this specific case

**General principle**: Push state down to the component that needs it. Don't lift state up unless multiple components need to share it.

### When React.memo and useCallback Are Actually Needed

Our refactor eliminated the need for React.memo in this case. But there are scenarios where you do need it:

#### Scenario 1: Expensive Rendering

If a component is computationally expensive to render:

```tsx
// Component that does heavy calculations
const ExpensiveChart = memo(function ExpensiveChart({ data }: { data: number[] }) {
  // Expensive calculations here
  const processedData = data.map(/* complex transformations */);
  
  return <canvas>{/* render chart */}</canvas>;
});
```

#### Scenario 2: Large Lists

When rendering many items that don't change often:

```tsx
// List of 1000 items
const ProductList = ({ products }: { products: Product[] }) => {
  const handleAddToCart = useCallback((productId: string) => {
    // Add to cart logic
  }, []);

  return (
    <div>
      {products.map(product => (
        <ProductCard 
          key={product.id}
          product={product}
          onAddToCart={handleAddToCart}  // Stable function reference
        />
      ))}
    </div>
  );
};

// Memoized to prevent re-render when parent re-renders
const ProductCard = memo(function ProductCard({ 
  product, 
  onAddToCart 
}: { 
  product: Product; 
  onAddToCart: (id: string) => void;
}) {
  return (
    <div>
      <h3>{product.name}</h3>
      <button onClick={() => onAddToCart(product.id)}>Add to Cart</button>
    </div>
  );
});
```

#### Scenario 3: Props That Are Objects or Arrays

When you must pass objects/arrays as props:

```tsx
function ParentComponent() {
  const [count, setCount] = useState(0);
  
  // Without useMemo, this creates a new object on every render
  const config = useMemo(() => ({
    theme: 'dark',
    fontSize: 14,
    showLineNumbers: true
  }), []);  // Empty deps = never changes

  return <CodeEditor config={config} />;
}

const CodeEditor = memo(function CodeEditor({ 
  config 
}: { 
  config: EditorConfig 
}) {
  // Expensive editor initialization
  return <div>{/* editor UI */}</div>;
});
```

### Common Failure Modes and Their Signatures

#### Symptom: React.memo doesn't prevent re-renders

**Browser behavior**:
Component still re-renders despite being wrapped in memo.

**Console pattern**:
```
Rendering MyComponent (props: { data: {...}, onClick: [Function] })
Rendering MyComponent (props: { data: {...}, onClick: [Function] })
```

**DevTools clues**:
- Profiler shows "Props changed" as reason
- Props appear identical but are different object references

**Root cause**: Props are objects, arrays, or functions recreated on each render

**Solution**: Use `useCallback` for functions, `useMemo` for objects/arrays, or redesign component boundaries

#### Symptom: useCallback dependencies cause infinite loops

**Browser behavior**:
Browser freezes, "Maximum update depth exceeded" error.

**Console pattern**:
```
Error: Maximum update depth exceeded. This can happen when a component 
repeatedly calls setState inside useEffect, or when useEffect has a 
dependency that changes on every render.
```

**Example**:

```tsx
// ❌ WRONG: handleClick depends on count, which changes, which recreates handleClick
const [count, setCount] = useState(0);

const handleClick = useCallback(() => {
  setCount(count + 1);  // ← Depends on count
}, [count]);  // ← count changes, so handleClick changes

useEffect(() => {
  // This effect runs when handleClick changes
  // But handleClick changes when count changes
  // And this effect might change count...
  // INFINITE LOOP
}, [handleClick]);
```

**Solution**: Use functional state updates:

```tsx
// ✓ CORRECT: No dependency on count
const handleClick = useCallback(() => {
  setCount(prev => prev + 1);  // ← Functional update
}, []);  // ← Empty deps, never changes
```

#### Symptom: Premature optimization makes code worse

**Browser behavior**:
Code works but is harder to read and maintain.

**Example**:

```tsx
// ❌ Unnecessary optimization
const MyComponent = memo(function MyComponent({ name }: { name: string }) {
  return <p>Hello, {name}</p>;
});

// This component is so simple that memo adds no value
// It just makes the code harder to understand
```

**Root cause**: Optimizing before measuring

**Solution**: Profile first, optimize second

### When to Apply These Optimizations

**React.memo**:
- Use for: Components that render frequently with the same props
- Optimizes for: Preventing unnecessary re-renders
- Avoid when: Component is simple and fast to render
- Measure: Use React DevTools Profiler to confirm it helps

**useCallback**:
- Use for: Functions passed to memoized child components
- Optimizes for: Stable function references
- Avoid when: Function is not passed to memoized children
- Measure: Check if removing it causes performance issues

**useMemo**:
- Use for: Expensive calculations or object/array props
- Optimizes for: Avoiding recalculation
- Avoid when: Calculation is cheap (< 1ms)
- Measure: Profile the calculation time

**Component Boundary Redesign**:
- Use for: State that only one component needs
- Optimizes for: Reducing re-render scope
- Prefer this over: memo/useCallback when possible
- Measure: Simplicity and maintainability

**General Rule**: Don't optimize until you have a performance problem. When you do optimize, measure before and after to confirm it helped.

## The Complete Journey - Chapter 5 Synthesis

## The Complete Journey: From Naive Lists to Optimized Rendering

Let's trace the evolution of our Activity Feed through all its iterations, examining what we learned at each stage.

### The Journey: From Problem to Solution

| Iteration | Problem                                  | Technique Applied                | Result                                | Performance Impact                |
| --------- | ---------------------------------------- | -------------------------------- | ------------------------------------- | --------------------------------- |
| 0         | No keys in list                          | None                             | Console warning, potential bugs       | Baseline                          |
| 1         | Added keys                               | `key={activity.id}`              | Warning gone, correct reconciliation  | No change                         |
| 2         | Wrong keys (index)                       | `key={index}`                    | State appears on wrong items          | No change                         |
| 3         | Fixed keys                               | `key={activity.id}` (corrected)  | State preserved correctly             | No change                         |
| 4         | No empty/error states                    | Conditional rendering patterns   | Better UX for edge cases              | No change                         |
| 5         | All items re-render on timer             | Passed `currentTime` prop        | Timestamps update, but inefficient    | 50 re-renders/min (45ms)          |
| 6         | Attempted React.memo                     | `memo(ActivityItem)`             | Still re-renders (props change)       | No improvement                    |
| 7         | Attempted useCallback                    | `useCallback(handleLike)`        | Still re-renders (currentTime object) | No improvement                    |
| 8         | Redesigned component boundaries          | Each item manages own timer      | Only changed items re-render          | 1 re-render/min (0.8ms) - 56x ✓  |
| 9         | Extracted state components               | Separate Loading/Error/Empty     | Cleaner code, better maintainability  | No change (code quality win)      |

### Final Implementation: Production-Ready Activity Feed

Here's our complete, optimized implementation:

```typescript
// src/types/activity.ts
export interface Activity {
  id: string;
  type: 'login' | 'purchase' | 'comment' | 'like' | 'share';
  description: string;
  timestamp: Date;
  metadata?: {
    amount?: number;
    productName?: string;
    targetUser?: string;
    likes?: number;
  };
}
```

```tsx
// src/components/ActivityFeed/LoadingState.tsx
export function LoadingState() {
  return (
    <div className="activity-loading">
      <div className="spinner" />
      <p>Loading activities...</p>
    </div>
  );
}
```

```tsx
// src/components/ActivityFeed/ErrorState.tsx
interface ErrorStateProps {
  error: string;
  onRetry: () => void;
}

export function ErrorState({ error, onRetry }: ErrorStateProps) {
  return (
    <div className="activity-error">
      <span className="error-icon">⚠️</span>
      <h3>Failed to load activities</h3>
      <p>{error}</p>
      <button onClick={onRetry}>Try Again</button>
    </div>
  );
}
```

```tsx
// src/components/ActivityFeed/EmptyState.tsx
interface EmptyStateProps {
  isFiltered?: boolean;
  filterType?: string;
  onClearFilter?: () => void;
}

export function EmptyState({ isFiltered, filterType, onClearFilter }: EmptyStateProps) {
  if (isFiltered) {
    return (
      <div className="activity-empty-filter">
        <p>No {filterType} activities found.</p>
        <button onClick={onClearFilter}>Show all activities</button>
      </div>
    );
  }

  return (
    <div className="activity-empty">
      <span className="empty-icon">📭</span>
      <h3>No activity yet</h3>
      <p>Your recent actions will appear here.</p>
    </div>
  );
}
```

```tsx
// src/components/ActivityItem.tsx - Final Version
import { useState, useEffect, memo } from 'react';
import { Activity } from '../types/activity';

interface ActivityItemProps {
  activity: Activity;
  onLike: (activityId: string) => void;
}

export const ActivityItem = memo(function ActivityItem({ 
  activity, 
  onLike 
}: ActivityItemProps) {
  const [isLiked, setIsLiked] = useState(false);
  const [likeCount, setLikeCount] = useState(activity.metadata?.likes || 0);
  const [relativeTime, setRelativeTime] = useState('');

  useEffect(() => {
    const updateTime = () => {
      const now = new Date();
      const diffMs = now.getTime() - new Date(activity.timestamp).getTime();
      const diffMins = Math.floor(diffMs / 60000);
      
      if (diffMins < 1) setRelativeTime('Just now');
      else if (diffMins < 60) setRelativeTime(`${diffMins}m ago`);
      else if (diffMins < 1440) setRelativeTime(`${Math.floor(diffMins / 60)}h ago`);
      else setRelativeTime(new Date(activity.timestamp).toLocaleDateString());
    };

    updateTime();
    const interval = setInterval(updateTime, 60000);
    return () => clearInterval(interval);
  }, [activity.timestamp]);

  const handleLike = () => {
    setIsLiked(!isLiked);
    setLikeCount(prev => isLiked ? prev - 1 : prev + 1);
    onLike(activity.id);
  };

  const getActivityIcon = (type: Activity['type']) => {
    const icons = {
      login: '🔐',
      purchase: '🛒',
      comment: '💬',
      like: '❤️',
      share: '🔄'
    };
    return icons[type];
  };

  return (
    <div className="activity-item">
      <span className="activity-icon">{getActivityIcon(activity.type)}</span>
      <div className="activity-content">
        <p className="activity-description">{activity.description}</p>
        {activity.metadata?.productName && (
          <span className="activity-meta">
            Product: {activity.metadata.productName}
          </span>
        )}
      </div>
      <div className="activity-actions">
        <button 
          onClick={handleLike}
          className={isLiked ? 'liked' : ''}
          aria-label={isLiked ? 'Unlike' : 'Like'}
        >
          {isLiked ? '❤️' : '🤍'} {likeCount}
        </button>
        <span className="activity-time">{relativeTime}</span>
      </div>
    </div>
  );
});
```

```tsx
// src/components/ActivityFeed.tsx - Final Version
import { useState, useEffect, useCallback } from 'react';
import { Activity } from '../types/activity';
import { ActivityItem } from './ActivityItem';
import { LoadingState } from './ActivityFeed/LoadingState';
import { ErrorState } from './ActivityFeed/ErrorState';
import { EmptyState } from './ActivityFeed/EmptyState';

type ActivityFilter = 'all' | Activity['type'];

export function ActivityFeed({ userId }: { userId: string }) {
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [filter, setFilter] = useState<ActivityFilter>('all');

  const fetchActivities = useCallback(() => {
    setIsLoading(true);
    setError(null);
    
    fetch(`/api/users/${userId}/activities`)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}: ${res.statusText}`);
        return res.json();
      })
      .then(data => {
        setActivities(data);
        setIsLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setIsLoading(false);
      });
  }, [userId]);

  useEffect(() => {
    fetchActivities();
  }, [fetchActivities]);

  const handleLike = useCallback((activityId: string) => {
    // In production, this would make an API call
    fetch(`/api/activities/${activityId}/like`, { method: 'POST' })
      .then(res => res.json())
      .then(data => {
        console.log('Liked activity:', data);
      })
      .catch(err => {
        console.error('Failed to like activity:', err);
      });
  }, []);

  const filteredActivities = filter === 'all' 
    ? activities 
    : activities.filter(a => a.type === filter);

  // Early returns for distinct states
  if (isLoading) {
    return (
      <div className="activity-feed">
        <LoadingState />
      </div>
    );
  }

  if (error) {
    return (
      <div className="activity-feed">
        <ErrorState error={error} onRetry={fetchActivities} />
      </div>
    );
  }

  if (activities.length === 0) {
    return (
      <div className="activity-feed">
        <EmptyState />
      </div>
    );
  }

  return (
    <div className="activity-feed">
      <div className="activity-header">
        <h2>Recent Activity</h2>
        <div className="activity-filters" role="tablist">
          <button 
            role="tab"
            aria-selected={filter === 'all'}
            onClick={() => setFilter('all')}
            className={filter === 'all' ? 'active' : ''}
          >
            All ({activities.length})
          </button>
          <button 
            role="tab"
            aria-selected={filter === 'purchase'}
            onClick={() => setFilter('purchase')}
            className={filter === 'purchase' ? 'active' : ''}
          >
            Purchases ({activities.filter(a => a.type === 'purchase').length})
          </button>
          <button 
            role="tab"
            aria-selected={filter === 'comment'}
            onClick={() => setFilter('comment')}
            className={filter === 'comment' ? 'active' : ''}
          >
            Comments ({activities.filter(a => a.type === 'comment').length})
          </button>
          <button 
            role="tab"
            aria-selected={filter === 'like'}
            onClick={() => setFilter('like')}
            className={filter === 'like' ? 'active' : ''}
          >
            Likes ({activities.filter(a => a.type === 'like').length})
          </button>
        </div>
      </div>
      
      {filteredActivities.length === 0 ? (
        <EmptyState 
          isFiltered 
          filterType={filter}
          onClearFilter={() => setFilter('all')}
        />
      ) : (
        <div className="activity-list" role="feed" aria-label="Activity feed">
          {filteredActivities.map(activity => (
            <ActivityItem 
              key={activity.id} 
              activity={activity}
              onLike={handleLike}
            />
          ))}
        </div>
      )}
    </div>
  );
}
```

### Decision Framework: List Rendering and Performance

Use this framework to make decisions about list rendering and optimization:

#### When to Use Keys

**Always use unique, stable keys for lists**:
- ✓ Use data IDs: `key={item.id}`
- ✓ Generate stable IDs: `key={`${item.type}-${item.timestamp}`}`
- ⚠️ Use index only if: List never reorders, items have no state, items never added/removed
- ❌ Never use: `Math.random()`, `Date.now()`, or any value that changes on each render

#### When to Use Conditional Rendering Patterns

**Early returns** (guard clauses):
- Use for: Mutually exclusive states (loading, error, empty, success)
- Best when: Each state needs completely different UI
- Example: Loading spinner vs. error message vs. data display

**Ternary operator**:
- Use for: Simple binary conditions
- Best when: Both branches are simple
- Avoid: Nesting more than one level

**Logical AND (&&)**:
- Use for: Showing/hiding single elements
- Best when: No "else" branch needed
- Watch out: Falsy values like `0` will render

**Object mapping**:
- Use for: 3+ distinct states
- Best when: States are represented by union types
- Example: Status indicators, theme variants

#### When to Optimize with React.memo

**Use React.memo when**:
1. Component renders frequently with same props
2. Component is expensive to render (> 5ms)
3. Component is in a large list (> 50 items)
4. Profiler shows it's a bottleneck

**Don't use React.memo when**:
1. Component is simple and fast (< 1ms)
2. Props change on every render anyway
3. You haven't measured a performance problem
4. Component rarely re-renders

#### When to Use useCallback

**Use useCallback when**:
1. Passing function to memoized child component
2. Function is a dependency of useEffect
3. Function is expensive to create
4. Profiler shows function recreation is a problem

**Don't use useCallback when**:
1. Function is not passed to memoized children
2. Function has no dependencies
3. You haven't measured a performance problem
4. It makes code harder to read

#### When to Redesign Component Boundaries

**Consider redesigning when**:
1. Parent re-renders cause many children to re-render
2. State is only used by one component
3. You're using many memo/useCallback to prevent re-renders
4. Component tree is deeply nested

**Redesign strategies**:
1. **Push state down**: Move state to the component that uses it
2. **Lift content up**: Pass children as props to avoid re-renders
3. **Split components**: Separate frequently-changing parts from static parts
4. **Use composition**: Combine small, focused components

### Lessons Learned

#### 1. Keys Are Not Optional

Keys are React's way of tracking list items across renders. Without proper keys:
- State appears on wrong items
- Performance degrades
- Animations break
- Focus is lost

**Always use stable, unique identifiers as keys.**

#### 2. Conditional Rendering Is About Clarity

Choose the pattern that makes your intent clearest:
- Early returns for mutually exclusive states
- Ternaries for simple binary conditions
- Logical AND for showing/hiding elements
- Object mapping for multiple states

**Optimize for readability first, performance second.**

#### 3. Optimization Is About Measurement

Don't optimize until you have evidence of a problem:
1. Use React DevTools Profiler to measure
2. Identify the bottleneck
3. Apply the appropriate technique
4. Measure again to confirm improvement

**Premature optimization makes code harder to maintain.**

#### 4. Component Boundaries Matter More Than Optimization Techniques

The best optimization is often redesigning component boundaries:
- Push state down to where it's used
- Avoid passing props that change frequently
- Let components manage their own concerns

**Good architecture beats clever optimization.**

#### 5. Reference Equality vs. Value Equality

Objects, arrays, and functions are compared by reference, not value:
- New object/array/function = new reference
- React.memo sees new reference as "changed"
- Use useCallback/useMemo to stabilize references

**Understand JavaScript's equality semantics to use React effectively.**

### What's Next

In Chapter 6, we'll tackle forms and validation. We'll see how the patterns we learned here—proper keys, conditional rendering, and performance optimization—apply to complex form interactions. We'll also introduce React Hook Form and Zod to handle form state and validation professionally.

Our Activity Feed is now production-ready:
- ✓ Proper keys for correct reconciliation
- ✓ Comprehensive error handling
- ✓ Optimized rendering (56x faster)
- ✓ Clean, maintainable code
- ✓ Accessible UI with ARIA attributes

This is the foundation for building any list-based UI in React.
