# Chapter 12: React Query: Server State Made Simple

## Client state vs. server state

## The Fundamental Confusion: Not All State Is Created Equal

Before we dive into React Query, we need to understand a critical distinction that most React developers miss: **client state and server state are fundamentally different problems that require different solutions.**

In Chapter 11, we built a task board using Zustand for global state management. It worked well for managing UI state—filters, selected items, modal visibility. But when we added server data (tasks fetched from an API), we encountered a cascade of problems that Zustand wasn't designed to solve.

Let's establish our reference implementation: a **Project Dashboard** that displays projects, their tasks, and team members. This will be our anchor example throughout this chapter, and we'll watch it fail in instructive ways before React Query saves us.

### Reference Implementation: Project Dashboard (Naive Approach)

Here's how most developers initially handle server data—using the same state management tools they use for client state:

```tsx
// src/components/ProjectDashboard.tsx
import { create } from 'zustand';
import { useEffect } from 'react';

interface Project {
  id: string;
  name: string;
  status: 'active' | 'completed' | 'archived';
  taskCount: number;
  teamSize: number;
}

interface DashboardStore {
  projects: Project[];
  isLoading: boolean;
  error: string | null;
  fetchProjects: () => Promise<void>;
  refreshProjects: () => Promise<void>;
}

// Using Zustand to manage server data (this will fail)
const useDashboardStore = create<DashboardStore>((set) => ({
  projects: [],
  isLoading: false,
  error: null,
  
  fetchProjects: async () => {
    set({ isLoading: true, error: null });
    try {
      const response = await fetch('/api/projects');
      if (!response.ok) throw new Error('Failed to fetch');
      const data = await response.json();
      set({ projects: data, isLoading: false });
    } catch (error) {
      set({ error: (error as Error).message, isLoading: false });
    }
  },
  
  refreshProjects: async () => {
    // Just call fetchProjects again
    const store = useDashboardStore.getState();
    await store.fetchProjects();
  }
}));

export function ProjectDashboard() {
  const { projects, isLoading, error, fetchProjects } = useDashboardStore();
  
  useEffect(() => {
    fetchProjects();
  }, [fetchProjects]);
  
  if (isLoading) return <div>Loading projects...</div>;
  if (error) return <div>Error: {error}</div>;
  
  return (
    <div className="dashboard">
      <h1>Projects ({projects.length})</h1>
      <div className="project-grid">
        {projects.map(project => (
          <ProjectCard key={project.id} project={project} />
        ))}
      </div>
    </div>
  );
}

function ProjectCard({ project }: { project: Project }) {
  return (
    <div className="project-card">
      <h3>{project.name}</h3>
      <p>Status: {project.status}</p>
      <p>Tasks: {project.taskCount}</p>
      <p>Team: {project.teamSize} members</p>
    </div>
  );
}
```

This looks reasonable. It loads data, shows loading states, handles errors. Ship it, right?

**Wrong.** Let's watch it fail.

### The Failure: Stale Data Everywhere

Open the app in two browser tabs. In Tab 1, the dashboard shows 5 projects. Switch to Tab 2—it also shows 5 projects. Now, in a third tab, add a new project through your admin panel. Switch back to Tab 1. Still shows 5 projects. Switch to Tab 2. Still shows 5 projects.

**The data is stale, and the user has no idea.**

Refresh the page manually. Now it shows 6 projects. But this is 2025—users shouldn't need to refresh pages.

### Diagnostic Analysis: Reading the Stale Data Problem

**Browser Behavior**:
- User sees outdated project count
- No indication that data might be stale
- Manual refresh required to see updates
- Data inconsistency across tabs

**Browser Console Output**:
```
(No errors—the code works as designed, which is the problem)
```

**React DevTools Evidence**:
- `ProjectDashboard` component state: `{ projects: [...5 items], isLoading: false }`
- State never updates after initial load
- No re-fetch mechanism triggered by time or user action

**Network Tab Analysis**:
- Single request to `/api/projects` on component mount
- No subsequent requests
- No polling, no refetch on focus, no background updates

**Let's parse this evidence**:

1. **What the user experiences**: Data appears correct but becomes increasingly outdated over time. No visual indication of staleness.

2. **What the console reveals**: Nothing—there are no errors because the code is working exactly as written.

3. **What DevTools shows**: State is set once and never updated. The component has no mechanism to know when data might be stale.

4. **Root cause identified**: We're treating server data like client data—set it once and forget it. But server data has a source of truth (the server) that can change independently of our application.

5. **Why the current approach can't solve this**: Zustand is designed for client state that the application controls. It has no concept of "freshness," "refetching," or "background updates."

6. **What we need**: A state management solution that understands server data is fundamentally different—it can become stale, needs periodic updates, and should refetch intelligently.

### The Failure: Cache Invalidation Hell

Let's add a feature: creating a new project. Users click "New Project," fill out a form, submit it. The project is created on the server. Now what?

```tsx
// src/components/CreateProjectForm.tsx
import { useDashboardStore } from './ProjectDashboard';

export function CreateProjectForm() {
  const refreshProjects = useDashboardStore(state => state.refreshProjects);
  
  const handleSubmit = async (formData: FormData) => {
    const response = await fetch('/api/projects', {
      method: 'POST',
      body: JSON.stringify({
        name: formData.get('name'),
        status: 'active'
      }),
      headers: { 'Content-Type': 'application/json' }
    });
    
    if (response.ok) {
      // Now we need to update the project list
      await refreshProjects(); // ← Manual cache invalidation
    }
  };
  
  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      handleSubmit(new FormData(e.currentTarget));
    }}>
      <input name="name" placeholder="Project name" required />
      <button type="submit">Create Project</button>
    </form>
  );
}
```

We manually call `refreshProjects()` after creating a project. This works... for this one form. But now imagine:

- Editing a project (need to refresh)
- Deleting a project (need to refresh)
- Archiving a project (need to refresh)
- Updating project status (need to refresh)
- Adding a team member (need to refresh project details)

Every mutation requires manual cache invalidation. Miss one, and your UI shows stale data. This is **cache invalidation hell**.

### Diagnostic Analysis: The Manual Invalidation Problem

**Browser Behavior**:
- After creating a project, user sees the new project appear (good)
- After editing a project in a modal, user closes modal—project list still shows old data (bad)
- After deleting a project, it still appears in the list until page refresh (bad)

**Browser Console Output**:
```
POST /api/projects 201 Created
(No automatic refetch—developer forgot to call refreshProjects)
```

**Code Evidence**:
```tsx
// Developer forgot to invalidate cache after edit
const handleEdit = async (projectId: string, updates: Partial<Project>) => {
  await fetch(`/api/projects/${projectId}`, {
    method: 'PATCH',
    body: JSON.stringify(updates)
  });
  // ← Missing: refreshProjects()
  closeModal();
};
```

**Let's parse this evidence**:

1. **What the user experiences**: Inconsistent behavior—sometimes the UI updates, sometimes it doesn't, depending on whether the developer remembered to invalidate the cache.

2. **What the code reveals**: Cache invalidation is manual, error-prone, and scattered across the codebase. Every mutation needs to know which queries to invalidate.

3. **Root cause identified**: We're manually managing the relationship between mutations and cached data. This doesn't scale.

4. **Why the current approach can't solve this**: Zustand has no concept of "queries" and "mutations" or automatic cache invalidation. It's just a state store.

5. **What we need**: A system that automatically tracks which data depends on which server resources and invalidates caches intelligently.

### The Failure: Loading States Everywhere

Let's add project details. Click a project card, open a modal showing full project details including tasks and team members.

```tsx
// src/components/ProjectDetailsModal.tsx
import { create } from 'zustand';
import { useEffect } from 'react';

interface ProjectDetails {
  id: string;
  name: string;
  description: string;
  tasks: Task[];
  team: TeamMember[];
}

interface DetailsStore {
  details: ProjectDetails | null;
  isLoading: boolean;
  error: string | null;
  fetchDetails: (projectId: string) => Promise<void>;
}

const useDetailsStore = create<DetailsStore>((set) => ({
  details: null,
  isLoading: false,
  error: null,
  
  fetchDetails: async (projectId: string) => {
    set({ isLoading: true, error: null });
    try {
      const response = await fetch(`/api/projects/${projectId}`);
      const data = await response.json();
      set({ details: data, isLoading: false });
    } catch (error) {
      set({ error: (error as Error).message, isLoading: false });
    }
  }
}));

export function ProjectDetailsModal({ projectId }: { projectId: string }) {
  const { details, isLoading, error, fetchDetails } = useDetailsStore();
  
  useEffect(() => {
    fetchDetails(projectId);
  }, [projectId, fetchDetails]);
  
  if (isLoading) return <div>Loading details...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!details) return null;
  
  return (
    <div className="modal">
      <h2>{details.name}</h2>
      <p>{details.description}</p>
      <h3>Tasks ({details.tasks.length})</h3>
      {/* ... */}
    </div>
  );
}
```

Now we have **two separate stores** managing related data. The project list is in `useDashboardStore`, project details are in `useDetailsStore`. They can get out of sync. Worse, every time you open the modal, you see "Loading details..." even if you just viewed this project 5 seconds ago.

### Diagnostic Analysis: The Redundant Loading Problem

**Browser Behavior**:
- User clicks Project A → sees loading spinner
- User closes modal, clicks Project A again → sees loading spinner again
- User clicks Project B → sees loading spinner
- User clicks Project A again → sees loading spinner yet again

**Network Tab Analysis**:
```
GET /api/projects/abc-123 200 OK (245ms)
GET /api/projects/abc-123 200 OK (238ms)  ← Same request, 10 seconds later
GET /api/projects/def-456 200 OK (251ms)
GET /api/projects/abc-123 200 OK (242ms)  ← Same request again, 20 seconds later
```

**React DevTools Evidence**:
- `useDetailsStore` state resets to `{ details: null, isLoading: true }` on every modal open
- No caching—previous data is discarded
- Component remounts trigger full refetch

**Let's parse this evidence**:

1. **What the user experiences**: Unnecessary loading spinners for data they just viewed. Feels slow and unresponsive.

2. **What the Network tab reveals**: Identical requests being made repeatedly for the same data within seconds.

3. **What DevTools shows**: State is reset on every fetch. No memory of previous data.

4. **Root cause identified**: We're not caching server data—we're just storing the most recent fetch result. Every new request starts from scratch.

5. **Why the current approach can't solve this**: Zustand doesn't have built-in caching, deduplication, or "freshness" concepts. We'd have to build all of this ourselves.

6. **What we need**: Intelligent caching that remembers previous fetches, serves cached data instantly, and refetches in the background when data might be stale.

### The Failure: Race Conditions

Let's add a search feature. User types in a search box, and we filter projects by name.

```tsx
// src/components/ProjectSearch.tsx
import { useState, useEffect } from 'react';

export function ProjectSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Project[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  
  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }
    
    setIsLoading(true);
    fetch(`/api/projects/search?q=${query}`)
      .then(res => res.json())
      .then(data => {
        setResults(data);
        setIsLoading(false);
      });
  }, [query]);
  
  return (
    <div>
      <input 
        value={query} 
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search projects..."
      />
      {isLoading && <div>Searching...</div>}
      <div>{results.map(project => <ProjectCard key={project.id} project={project} />)}</div>
    </div>
  );
}
```

User types "react" quickly. The effect fires for "r", "re", "rea", "reac", "react". Five requests are in flight. The request for "rea" takes 500ms and returns 20 results. The request for "react" takes 200ms and returns 5 results. Which response arrives first?

**The "react" response arrives first** (200ms), sets results to 5 projects. Then the "rea" response arrives (500ms), sets results to 20 projects. The user searched for "react" but sees results for "rea".

### Diagnostic Analysis: The Race Condition

**Browser Behavior**:
- User types "react" in search box
- Sees 5 results briefly
- Then sees 20 results (wrong!)
- Search box still shows "react" but results are for "rea"

**Browser Console Output**:
```
(No errors—race conditions are silent)
```

**Network Tab Analysis**:
```
GET /api/projects/search?q=r      200 OK (450ms)
GET /api/projects/search?q=re     200 OK (380ms)
GET /api/projects/search?q=rea    200 OK (500ms)  ← Slowest, arrives last
GET /api/projects/search?q=reac   200 OK (290ms)
GET /api/projects/search?q=react  200 OK (200ms)  ← Fastest, arrives first
```

**Let's parse this evidence**:

1. **What the user experiences**: Search results don't match the search query. Confusing and broken-feeling UI.

2. **What the Network tab reveals**: Multiple overlapping requests with different response times. Later requests can complete before earlier ones.

3. **Root cause identified**: We're not canceling previous requests or tracking which request is the "current" one. The last response to arrive wins, regardless of which query it was for.

4. **Why the current approach can't solve this**: We'd need to manually implement request cancellation with AbortController, track request IDs, and ignore stale responses. This is complex and error-prone.

5. **What we need**: Automatic request deduplication and cancellation. Only the most recent query should update the UI.

## The Conceptual Divide: Client State vs. Server State

Let's step back and understand why all these problems exist.

### Client State

**Client state** is data that your application owns and controls:

- UI state: is the modal open? which tab is selected?
- Form state: what's in the input fields?
- User preferences: theme, language, sidebar collapsed?

**Characteristics**:
- **Synchronous**: Changes happen instantly in memory
- **Authoritative**: Your app is the source of truth
- **Persistent**: Stays the same until you change it
- **Predictable**: You control all mutations

**Tools**: `useState`, `useReducer`, Zustand, Redux

### Server State

**Server state** is data that your application does NOT own:

- Database records: projects, tasks, users
- API responses: search results, analytics data
- Real-time data: notifications, live updates

**Characteristics**:
- **Asynchronous**: Requires network requests
- **Remote**: Server is the source of truth
- **Potentially stale**: Can change without your knowledge
- **Shared**: Other users/systems can modify it

**Problems**:
- **Staleness**: Data becomes outdated
- **Caching**: Should we refetch or use cached data?
- **Deduplication**: Multiple components requesting the same data
- **Invalidation**: When mutations happen, what needs to refetch?
- **Race conditions**: Overlapping requests with different response times
- **Loading states**: Per-request or global?
- **Error handling**: Retry logic, error boundaries
- **Optimistic updates**: Show changes before server confirms

**Wrong tool**: Zustand, Redux (designed for client state)

**Right tool**: React Query (designed specifically for server state)

## When to Apply: State Classification Decision Tree

Before choosing a state management solution, classify your state:

**Is this data fetched from a server?**
- **No** → Client state → Use `useState`, `useReducer`, or Zustand
- **Yes** → Continue...

**Does this data change on the server without your app's knowledge?**
- **No** (e.g., static configuration) → Client state is fine
- **Yes** → Server state → Use React Query

**Do multiple components need this data?**
- **No** → Local state with `useState` + `useEffect`
- **Yes** → React Query (automatic deduplication)

**Does this data need to stay fresh?**
- **No** (e.g., historical data) → Simple fetch + cache
- **Yes** → React Query (automatic refetching)

**Do mutations affect this data?**
- **No** → Simple fetch might suffice
- **Yes** → React Query (automatic invalidation)

**Summary**: If you're fetching data from an API and that data can change on the server, you almost certainly want React Query, not Zustand or Redux.

## TanStack Query (React Query) fundamentals

## Iteration 1: Introducing React Query

Let's fix our Project Dashboard using React Query (officially called TanStack Query, but everyone still calls it React Query).

### Setup

First, install the library:

```bash
npm install @tanstack/react-query
```

Set up the QueryClient and provider:

```tsx
// src/App.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

// Create a client
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes
      gcTime: 1000 * 60 * 10,    // 10 minutes (formerly cacheTime)
      retry: 1,
      refetchOnWindowFocus: true,
    },
  },
});

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <ProjectDashboard />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

**What we just did**:

- **QueryClient**: The central cache manager. Stores all query results, manages refetching, handles invalidation.
- **QueryClientProvider**: Makes the client available to all components via React context.
- **ReactQueryDevtools**: A dev panel showing all queries, their status, cached data, and refetch controls. Essential for debugging.

**Configuration explained**:

- `staleTime: 5 minutes`: Data is considered "fresh" for 5 minutes. Fresh data won't refetch automatically.
- `gcTime: 10 minutes`: Cached data is kept in memory for 10 minutes after the last component using it unmounts.
- `retry: 1`: If a request fails, retry once before showing an error.
- `refetchOnWindowFocus: true`: When user switches back to the tab, refetch stale data automatically.

### Iteration 1: Converting to useQuery

Now let's rewrite our ProjectDashboard using React Query:

```tsx
// src/components/ProjectDashboard.tsx
import { useQuery } from '@tanstack/react-query';

interface Project {
  id: string;
  name: string;
  status: 'active' | 'completed' | 'archived';
  taskCount: number;
  teamSize: number;
}

// Fetch function (pure, no state management)
async function fetchProjects(): Promise<Project[]> {
  const response = await fetch('/api/projects');
  if (!response.ok) {
    throw new Error('Failed to fetch projects');
  }
  return response.json();
}

export function ProjectDashboard() {
  // useQuery replaces useState + useEffect + error handling
  const { data: projects, isLoading, error } = useQuery({
    queryKey: ['projects'],
    queryFn: fetchProjects,
  });
  
  if (isLoading) return <div>Loading projects...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  return (
    <div className="dashboard">
      <h1>Projects ({projects?.length ?? 0})</h1>
      <div className="project-grid">
        {projects?.map(project => (
          <ProjectCard key={project.id} project={project} />
        ))}
      </div>
    </div>
  );
}

function ProjectCard({ project }: { project: Project }) {
  return (
    <div className="project-card">
      <h3>{project.name}</h3>
      <p>Status: {project.status}</p>
      <p>Tasks: {project.taskCount}</p>
      <p>Team: {project.teamSize} members</p>
    </div>
  );
}
```

**What changed**:

1. **No more Zustand store**: All state management is handled by React Query.
2. **Pure fetch function**: `fetchProjects` is just a function that returns a promise. No state, no side effects.
3. **useQuery hook**: Replaces `useState` + `useEffect` + error handling + loading states.
4. **queryKey**: A unique identifier for this query. React Query uses this for caching and deduplication.
5. **queryFn**: The function that fetches the data.

### Verification: The Stale Data Problem Is Solved

Open the app in two tabs. Add a project in a third tab. Now switch back to Tab 1. **Within 5 seconds, the new project appears automatically.**

**Browser Console Output**:
```
[React Query] Query ['projects'] is stale, refetching...
[React Query] Query ['projects'] fetched successfully
```

**React Query DevTools**:
- Query: `['projects']`
- Status: `success`
- Data Age: `2.3s`
- Last Updated: `2 seconds ago`
- Observers: `1` (one component using this data)

**Network Tab**:
```
GET /api/projects 200 OK (245ms)  ← Initial load
(User switches tabs)
GET /api/projects 200 OK (238ms)  ← Automatic refetch on focus
```

**What happened**:

1. Initial load fetches projects
2. Data is cached with key `['projects']`
3. After 5 minutes, data becomes "stale" (but still shown to user)
4. When user switches back to the tab, React Query sees stale data and refetches automatically
5. New data arrives, cache updates, component re-renders with fresh data

**No manual refresh required. No stale data. It just works.**

### The Query Key: React Query's Secret Weapon

The `queryKey` is not just an identifier—it's a **dependency array for your data**.

```tsx
// Simple key
useQuery({
  queryKey: ['projects'],
  queryFn: fetchProjects,
});

// Key with parameters
useQuery({
  queryKey: ['project', projectId],
  queryFn: () => fetchProject(projectId),
});

// Key with filters
useQuery({
  queryKey: ['projects', { status: 'active', search: query }],
  queryFn: () => fetchProjects({ status: 'active', search: query }),
});
```

**Rules**:

1. **Unique keys for unique data**: Different keys = different cache entries
2. **Include all variables**: If your fetch function uses a variable, include it in the key
3. **Stable serialization**: Objects in keys are compared by value, not reference

**Why this matters**:

- React Query automatically deduplicates requests with the same key
- Changing the key triggers a new fetch
- Invalidating a key refetches all queries with that key

### Iteration 2: Project Details with Automatic Caching

Let's fix the project details modal:

```tsx
// src/components/ProjectDetailsModal.tsx
import { useQuery } from '@tanstack/react-query';

interface ProjectDetails {
  id: string;
  name: string;
  description: string;
  tasks: Task[];
  team: TeamMember[];
}

async function fetchProjectDetails(projectId: string): Promise<ProjectDetails> {
  const response = await fetch(`/api/projects/${projectId}`);
  if (!response.ok) throw new Error('Failed to fetch project details');
  return response.json();
}

export function ProjectDetailsModal({ projectId }: { projectId: string }) {
  const { data: details, isLoading, error } = useQuery({
    queryKey: ['project', projectId],
    queryFn: () => fetchProjectDetails(projectId),
  });
  
  if (isLoading) return <div>Loading details...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!details) return null;
  
  return (
    <div className="modal">
      <h2>{details.name}</h2>
      <p>{details.description}</p>
      <h3>Tasks ({details.tasks.length})</h3>
      <ul>
        {details.tasks.map(task => (
          <li key={task.id}>{task.title}</li>
        ))}
      </ul>
      <h3>Team ({details.team.length})</h3>
      <ul>
        {details.team.map(member => (
          <li key={member.id}>{member.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Verification: Caching Works

**Test sequence**:

1. Click Project A → See loading spinner → Details appear
2. Close modal
3. Click Project A again → **Details appear instantly, no loading spinner**
4. Click Project B → See loading spinner → Details appear
5. Click Project A again → **Still instant**

**React Query DevTools**:
```
Query ['project', 'abc-123']:
  Status: success
  Data Age: 45s
  Fetch Status: idle
  Observers: 0 (modal closed, but data still cached)

Query ['project', 'def-456']:
  Status: success
  Data Age: 12s
  Fetch Status: idle
  Observers: 0
```

**Network Tab**:
```
GET /api/projects/abc-123 200 OK (245ms)  ← First open
(No request on second open—served from cache)
GET /api/projects/def-456 200 OK (251ms)  ← First open of Project B
(No request on third open of Project A—still cached)
```

**What happened**:

1. First open of Project A fetches data and caches it with key `['project', 'abc-123']`
2. Close modal—component unmounts, but cache persists
3. Second open of Project A—React Query finds cached data, serves it instantly
4. After 5 minutes (staleTime), data becomes stale but is still shown
5. React Query refetches in the background, updates cache when response arrives

**No redundant requests. Instant UI. Automatic background updates.**

### Iteration 3: Search with Automatic Deduplication

Let's fix the search race condition:

```tsx
// src/components/ProjectSearch.tsx
import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';

async function searchProjects(query: string): Promise<Project[]> {
  const response = await fetch(`/api/projects/search?q=${query}`);
  if (!response.ok) throw new Error('Search failed');
  return response.json();
}

export function ProjectSearch() {
  const [query, setQuery] = useState('');
  
  const { data: results, isLoading } = useQuery({
    queryKey: ['projects', 'search', query],
    queryFn: () => searchProjects(query),
    enabled: query.length > 0, // Only run query if there's a search term
  });
  
  return (
    <div>
      <input 
        value={query} 
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search projects..."
      />
      {isLoading && <div>Searching...</div>}
      <div>
        {results?.map(project => (
          <ProjectCard key={project.id} project={project} />
        ))}
      </div>
    </div>
  );
}
```

### Verification: Race Conditions Eliminated

**Test sequence**:

1. Type "react" quickly
2. See loading indicator
3. Results appear for "react" (5 projects)
4. Results never change to show "rea" results

**React Query DevTools**:
```
Query ['projects', 'search', 'r']:
  Status: success (cancelled)
  
Query ['projects', 'search', 're']:
  Status: success (cancelled)
  
Query ['projects', 'search', 'rea']:
  Status: success (cancelled)
  
Query ['projects', 'search', 'reac']:
  Status: success (cancelled)
  
Query ['projects', 'search', 'react']:
  Status: success
  Data Age: 1.2s
  Observers: 1
```

**Network Tab**:
```
GET /api/projects/search?q=r      (cancelled)
GET /api/projects/search?q=re     (cancelled)
GET /api/projects/search?q=rea    (cancelled)
GET /api/projects/search?q=reac   (cancelled)
GET /api/projects/search?q=react  200 OK (200ms)
```

**What happened**:

1. Each keystroke changes the query key: `['projects', 'search', 'r']`, `['projects', 'search', 're']`, etc.
2. React Query sees the key changed and **cancels the previous request**
3. Only the final query (`'react'`) completes
4. Results always match the current query

**No race conditions. No stale results. Automatic request cancellation.**

### Understanding Query Status

React Query provides detailed status information:

```tsx
const { 
  data,           // The data (undefined until first successful fetch)
  error,          // Error object if query failed
  isLoading,      // true if first fetch is in progress
  isFetching,     // true if any fetch is in progress (including background refetch)
  isError,        // true if query failed
  isSuccess,      // true if query succeeded
  status,         // 'pending' | 'error' | 'success'
  fetchStatus,    // 'fetching' | 'paused' | 'idle'
} = useQuery({
  queryKey: ['projects'],
  queryFn: fetchProjects,
});
```

**Key distinctions**:

- **isLoading**: First fetch in progress, no data yet. Show skeleton/spinner.
- **isFetching**: Any fetch in progress, might have cached data. Show subtle indicator.
- **isError**: Query failed. Show error message.
- **isSuccess**: Query succeeded. Show data.

**Common pattern**:

```tsx
function ProjectDashboard() {
  const { data, isLoading, isFetching, error } = useQuery({
    queryKey: ['projects'],
    queryFn: fetchProjects,
  });
  
  if (isLoading) {
    // First load, no data yet
    return <div>Loading projects...</div>;
  }
  
  if (error) {
    return <div>Error: {error.message}</div>;
  }
  
  return (
    <div>
      {isFetching && <div className="refetch-indicator">Updating...</div>}
      <h1>Projects ({data?.length ?? 0})</h1>
      {/* Show cached data while refetching in background */}
      <div className="project-grid">
        {data?.map(project => <ProjectCard key={project.id} project={project} />)}
      </div>
    </div>
  );
}
```

This pattern shows cached data immediately while displaying a subtle indicator during background refetches. Much better UX than showing a loading spinner every time.

## When to Apply: Query Configuration Decision Framework

React Query has many configuration options. Here's when to use each:

### staleTime

**What it controls**: How long data is considered "fresh" before React Query will refetch it.

**When to use short staleTime (0-30 seconds)**:
- Real-time data: notifications, live dashboards
- Frequently changing data: stock prices, sports scores
- Critical accuracy: financial data, inventory counts

**When to use medium staleTime (1-5 minutes)**:
- User-generated content: posts, comments, profiles
- Search results
- Analytics data

**When to use long staleTime (10+ minutes)**:
- Static content: documentation, help articles
- Configuration data: app settings, feature flags
- Reference data: country lists, categories

**Default**: 0 (always stale, refetch on every mount)

### gcTime (formerly cacheTime)

**What it controls**: How long unused data stays in cache after all components using it unmount.

**When to use short gcTime (1-5 minutes)**:
- Large datasets that consume memory
- Data that changes frequently
- One-time views (e.g., confirmation pages)

**When to use long gcTime (10-30 minutes)**:
- Data users navigate back to frequently
- Expensive queries (slow API, complex computation)
- Data that doesn't change often

**Default**: 5 minutes

### refetchOnWindowFocus

**What it controls**: Whether to refetch stale queries when user switches back to the tab.

**When to enable (true)**:
- Multi-tab workflows
- Data that might change while user is away
- Collaborative applications

**When to disable (false)**:
- Single-page workflows
- Data that rarely changes
- Expensive queries

**Default**: true

### refetchOnReconnect

**What it controls**: Whether to refetch stale queries when internet connection is restored.

**When to enable (true)**:
- Mobile applications
- Offline-capable apps
- Real-time data

**When to disable (false)**:
- Desktop-only apps with stable connections
- Static content

**Default**: true

### retry

**What it controls**: How many times to retry failed queries.

**When to use 0 retries**:
- User input errors (400 Bad Request)
- Authentication errors (401 Unauthorized)
- Not found errors (404)

**When to use 1-2 retries**:
- Network errors (timeout, connection refused)
- Server errors (500, 503)
- Rate limiting (429)

**When to use 3+ retries**:
- Critical data that must load
- Flaky APIs
- Background sync operations

**Default**: 3

### enabled

**What it controls**: Whether the query should run at all.

**When to use**:
- Conditional queries: only fetch if user is authenticated
- Dependent queries: only fetch details after list loads
- Search: only fetch if query is not empty
- Lazy loading: only fetch when user clicks "Load more"

**Example**:

```tsx
// Only fetch project details if projectId exists
const { data } = useQuery({
  queryKey: ['project', projectId],
  queryFn: () => fetchProject(projectId),
  enabled: !!projectId, // Convert to boolean
});

// Only fetch if user is authenticated
const { data } = useQuery({
  queryKey: ['user', 'profile'],
  queryFn: fetchUserProfile,
  enabled: isAuthenticated,
});

// Only fetch if search query is long enough
const { data } = useQuery({
  queryKey: ['search', query],
  queryFn: () => search(query),
  enabled: query.length >= 3,
});
```

## Queries, mutations, and invalidation

## The Failure: Manual Cache Invalidation Doesn't Scale

Remember our create project form? We manually called `refreshProjects()` after creating a project. Let's see what happens when we have multiple mutations:

```tsx
// src/components/ProjectActions.tsx
import { useQuery } from '@tanstack/react-query';

export function ProjectActions({ projectId }: { projectId: string }) {
  const { data: projects, refetch } = useQuery({
    queryKey: ['projects'],
    queryFn: fetchProjects,
  });
  
  const handleArchive = async () => {
    await fetch(`/api/projects/${projectId}/archive`, { method: 'POST' });
    refetch(); // ← Manual invalidation
  };
  
  const handleDelete = async () => {
    await fetch(`/api/projects/${projectId}`, { method: 'DELETE' });
    refetch(); // ← Manual invalidation
  };
  
  const handleUpdateStatus = async (status: string) => {
    await fetch(`/api/projects/${projectId}`, {
      method: 'PATCH',
      body: JSON.stringify({ status }),
    });
    refetch(); // ← Manual invalidation
  };
  
  return (
    <div>
      <button onClick={handleArchive}>Archive</button>
      <button onClick={handleDelete}>Delete</button>
      <button onClick={() => handleUpdateStatus('completed')}>
        Mark Complete
      </button>
    </div>
  );
}
```

**Problems**:

1. **Scattered invalidation logic**: Every mutation needs to know which queries to refetch
2. **Easy to forget**: Developer adds a new mutation, forgets to invalidate
3. **Over-fetching**: Refetching entire list when only one project changed
4. **Coupling**: Mutation components need to know about query keys

### Diagnostic Analysis: The Manual Invalidation Problem

**Browser Behavior**:
- User archives a project
- Project list refetches (good)
- User opens project details modal
- Details still show "active" status (bad—forgot to invalidate details query)

**React Query DevTools**:
```
Query ['projects']:
  Status: success
  Data Age: 0.5s (just refetched)
  
Query ['project', 'abc-123']:
  Status: success
  Data Age: 45s (stale, not refetched)
  Data: { status: 'active' } ← Wrong!
```

**Let's parse this evidence**:

1. **What the user experiences**: Inconsistent data—list shows archived project, but details show it as active.

2. **What DevTools reveals**: Only the `['projects']` query was refetched. The `['project', 'abc-123']` query is stale.

3. **Root cause identified**: We manually refetched the list query but forgot to refetch the details query.

4. **Why the current approach can't solve this**: Manual invalidation requires developers to remember all related queries. This doesn't scale.

5. **What we need**: Automatic cache invalidation based on relationships between queries.

## Iteration 4: Mutations with Automatic Invalidation

React Query provides `useMutation` for data modifications and automatic cache invalidation:

```tsx
// src/components/CreateProjectForm.tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

interface CreateProjectData {
  name: string;
  description: string;
  status: 'active' | 'completed' | 'archived';
}

async function createProject(data: CreateProjectData): Promise<Project> {
  const response = await fetch('/api/projects', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to create project');
  return response.json();
}

export function CreateProjectForm() {
  const queryClient = useQueryClient();
  
  const mutation = useMutation({
    mutationFn: createProject,
    onSuccess: () => {
      // Invalidate and refetch projects list
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });
  
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    mutation.mutate({
      name: formData.get('name') as string,
      description: formData.get('description') as string,
      status: 'active',
    });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="name" placeholder="Project name" required />
      <textarea name="description" placeholder="Description" />
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create Project'}
      </button>
      {mutation.isError && (
        <div className="error">Error: {mutation.error.message}</div>
      )}
      {mutation.isSuccess && (
        <div className="success">Project created!</div>
      )}
    </form>
  );
}
```

**What changed**:

1. **useMutation**: Handles the mutation lifecycle (pending, success, error)
2. **mutationFn**: The function that performs the mutation
3. **onSuccess**: Callback that runs after successful mutation
4. **queryClient.invalidateQueries**: Marks queries as stale and triggers refetch
5. **mutation.mutate**: Triggers the mutation with data

### Verification: Automatic Invalidation Works

**Test sequence**:

1. Open project list (5 projects)
2. Submit create form
3. See "Creating..." button state
4. Project list automatically updates to show 6 projects
5. No manual refresh needed

**React Query DevTools**:
```
Mutation:
  Status: success
  Variables: { name: "New Project", ... }
  
Query ['projects']:
  Status: success
  Data Age: 0.2s (just refetched)
  Invalidated: true
```

**Network Tab**:
```
POST /api/projects 201 Created (245ms)
GET /api/projects 200 OK (180ms)  ← Automatic refetch after invalidation
```

**What happened**:

1. User submits form
2. `mutation.mutate()` calls `createProject()`
3. Request succeeds, `onSuccess` callback runs
4. `queryClient.invalidateQueries({ queryKey: ['projects'] })` marks the query as stale
5. React Query automatically refetches the query
6. Component re-renders with new data

**No manual refetch. Automatic cache invalidation. It just works.**

### Iteration 5: Invalidating Multiple Related Queries

When you archive a project, both the list and the project details need to update:

```tsx
// src/components/ProjectActions.tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

async function archiveProject(projectId: string): Promise<void> {
  const response = await fetch(`/api/projects/${projectId}/archive`, {
    method: 'POST',
  });
  if (!response.ok) throw new Error('Failed to archive project');
}

export function ProjectActions({ projectId }: { projectId: string }) {
  const queryClient = useQueryClient();
  
  const archiveMutation = useMutation({
    mutationFn: () => archiveProject(projectId),
    onSuccess: () => {
      // Invalidate all queries that start with ['projects']
      queryClient.invalidateQueries({ queryKey: ['projects'] });
      // Invalidate this specific project's details
      queryClient.invalidateQueries({ queryKey: ['project', projectId] });
    },
  });
  
  return (
    <button 
      onClick={() => archiveMutation.mutate()}
      disabled={archiveMutation.isPending}
    >
      {archiveMutation.isPending ? 'Archiving...' : 'Archive Project'}
    </button>
  );
}
```

### Verification: Multiple Queries Invalidated

**Test sequence**:

1. Open project details modal for Project A
2. Click "Archive Project"
3. Modal shows "Archiving..." button
4. Project list updates (Project A now shows "archived" status)
5. Project details modal updates (status changes to "archived")
6. Both queries refetched automatically

**React Query DevTools**:
```
Query ['projects']:
  Status: success
  Data Age: 0.3s
  Invalidated: true
  
Query ['project', 'abc-123']:
  Status: success
  Data Age: 0.3s
  Invalidated: true
```

**Network Tab**:
```
POST /api/projects/abc-123/archive 200 OK (245ms)
GET /api/projects 200 OK (180ms)  ← List refetch
GET /api/projects/abc-123 200 OK (190ms)  ← Details refetch
```

**What happened**:

1. Archive mutation succeeds
2. `invalidateQueries({ queryKey: ['projects'] })` invalidates:
   - `['projects']` (exact match)
   - `['projects', 'search', 'react']` (starts with `['projects']`)
   - Any other query starting with `['projects']`
3. `invalidateQueries({ queryKey: ['project', projectId] })` invalidates:
   - `['project', 'abc-123']` (exact match)
4. All invalidated queries refetch automatically
5. All components using those queries re-render with fresh data

**No manual coordination. Automatic cascade. Consistent data everywhere.**

### Query Key Patterns for Invalidation

React Query matches query keys hierarchically:

```tsx
// Invalidate ALL queries
queryClient.invalidateQueries();

// Invalidate all project-related queries
queryClient.invalidateQueries({ queryKey: ['projects'] });
// Matches: ['projects'], ['projects', 'search', 'react'], ['projects', { status: 'active' }]

// Invalidate only the exact query
queryClient.invalidateQueries({ queryKey: ['projects'], exact: true });
// Matches: ['projects'] only

// Invalidate a specific project
queryClient.invalidateQueries({ queryKey: ['project', projectId] });
// Matches: ['project', 'abc-123'] only

// Invalidate all projects with a specific status
queryClient.invalidateQueries({ 
  queryKey: ['projects'],
  predicate: (query) => {
    const [, filters] = query.queryKey;
    return filters?.status === 'active';
  }
});
```

**Best practices**:

1. **Use hierarchical keys**: `['projects']` → `['projects', 'search']` → `['projects', 'search', query]`
2. **Invalidate broadly**: `invalidateQueries({ queryKey: ['projects'] })` catches all project queries
3. **Be specific when needed**: `invalidateQueries({ queryKey: ['project', id], exact: true })`

### Iteration 6: Delete Mutation with Cache Removal

When you delete a project, invalidating isn't enough—you need to remove it from the cache:

```tsx
// src/components/ProjectActions.tsx
async function deleteProject(projectId: string): Promise<void> {
  const response = await fetch(`/api/projects/${projectId}`, {
    method: 'DELETE',
  });
  if (!response.ok) throw new Error('Failed to delete project');
}

export function ProjectActions({ projectId }: { projectId: string }) {
  const queryClient = useQueryClient();
  
  const deleteMutation = useMutation({
    mutationFn: () => deleteProject(projectId),
    onSuccess: () => {
      // Remove this project from cache
      queryClient.removeQueries({ queryKey: ['project', projectId] });
      // Invalidate list to refetch without this project
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });
  
  return (
    <button 
      onClick={() => {
        if (confirm('Delete this project?')) {
          deleteMutation.mutate();
        }
      }}
      disabled={deleteMutation.isPending}
    >
      {deleteMutation.isPending ? 'Deleting...' : 'Delete Project'}
    </button>
  );
}
```

**What's different**:

- **removeQueries**: Completely removes the query from cache (not just invalidates)
- **Why**: Deleted data shouldn't be cached—it no longer exists on the server

### Mutation Status and Error Handling

Mutations provide detailed status information:

```tsx
const mutation = useMutation({
  mutationFn: createProject,
  onSuccess: (data, variables, context) => {
    // data: The response from the mutation
    // variables: The data passed to mutate()
    // context: Value returned from onMutate
    console.log('Created project:', data);
  },
  onError: (error, variables, context) => {
    // error: The error object
    // variables: The data passed to mutate()
    // context: Value returned from onMutate
    console.error('Failed to create project:', error);
  },
  onSettled: (data, error, variables, context) => {
    // Runs after success or error
    // Useful for cleanup
  },
});

// In component
const {
  mutate,       // Trigger the mutation
  mutateAsync,  // Trigger and return a promise
  isPending,    // Mutation in progress
  isError,      // Mutation failed
  isSuccess,    // Mutation succeeded
  error,        // Error object
  data,         // Response data
  reset,        // Reset mutation state
} = mutation;
```

**Common pattern for form submission**:

```tsx
export function CreateProjectForm() {
  const queryClient = useQueryClient();
  const [formData, setFormData] = useState({ name: '', description: '' });
  
  const mutation = useMutation({
    mutationFn: createProject,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['projects'] });
      setFormData({ name: '', description: '' }); // Reset form
    },
  });
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    mutation.mutate(formData);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={formData.name}
        onChange={(e) => setFormData({ ...formData, name: e.target.value })}
        placeholder="Project name"
        required
      />
      <textarea
        value={formData.description}
        onChange={(e) => setFormData({ ...formData, description: e.target.value })}
        placeholder="Description"
      />
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create Project'}
      </button>
      {mutation.isError && (
        <div className="error">
          Error: {mutation.error.message}
          <button onClick={() => mutation.reset()}>Dismiss</button>
        </div>
      )}
      {mutation.isSuccess && (
        <div className="success">
          Project created successfully!
          <button onClick={() => mutation.reset()}>Create Another</button>
        </div>
      )}
    </form>
  );
}
```

## When to Apply: Invalidation Strategy Decision Framework

### Invalidate vs. Remove vs. Update

**Use `invalidateQueries` when**:
- Data changed on the server
- You want to refetch to get the latest version
- Multiple queries might be affected
- Example: After creating/updating a project

**Use `removeQueries` when**:
- Data was deleted on the server
- Cached data is no longer valid
- You don't want stale data shown even briefly
- Example: After deleting a project

**Use `setQueryData` when**:
- You know exactly what the new data should be
- You want to update cache without refetching
- Optimistic updates (covered in next section)
- Example: After toggling a boolean field

### Invalidation Scope

**Invalidate broadly** (recommended):
```tsx
queryClient.invalidateQueries({ queryKey: ['projects'] });
```
- Catches all related queries
- Ensures consistency
- Slight over-fetching is acceptable

**Invalidate narrowly** (when performance matters):
```tsx
queryClient.invalidateQueries({ queryKey: ['project', projectId], exact: true });
```
- Only refetches specific query
- Reduces network traffic
- Risk: Related queries might be stale

**Invalidate selectively** (advanced):
```tsx
queryClient.invalidateQueries({
  predicate: (query) => {
    // Custom logic to determine which queries to invalidate
    return query.queryKey[0] === 'projects' && query.state.data?.status === 'active';
  }
});
```
- Fine-grained control
- Complex logic
- Use sparingly

### Timing: When to Invalidate

**Immediate invalidation** (default):
```tsx
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: ['projects'] });
}
```
- Refetch happens immediately
- User sees loading state briefly
- Ensures fresh data

**Delayed invalidation**:
```tsx
onSuccess: () => {
  setTimeout(() => {
    queryClient.invalidateQueries({ queryKey: ['projects'] });
  }, 1000);
}
```
- Give user time to see success message
- Refetch happens in background
- Better UX for non-critical updates

**Conditional invalidation**:
```tsx
onSuccess: (data) => {
  if (data.affectsOtherProjects) {
    queryClient.invalidateQueries({ queryKey: ['projects'] });
  }
}
```
- Only invalidate when necessary
- Reduces unnecessary refetches
- Requires server to indicate impact

## Optimistic updates

## The Failure: Slow Feedback on Mutations

Our current implementation works, but there's a UX problem. Watch what happens when you toggle a project's favorite status:

```tsx
// src/components/ProjectCard.tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

async function toggleFavorite(projectId: string, isFavorite: boolean): Promise<void> {
  const response = await fetch(`/api/projects/${projectId}/favorite`, {
    method: 'POST',
    body: JSON.stringify({ isFavorite }),
  });
  if (!response.ok) throw new Error('Failed to toggle favorite');
}

export function ProjectCard({ project }: { project: Project }) {
  const queryClient = useQueryClient();
  
  const toggleMutation = useMutation({
    mutationFn: (isFavorite: boolean) => toggleFavorite(project.id, isFavorite),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });
  
  return (
    <div className="project-card">
      <h3>{project.name}</h3>
      <button
        onClick={() => toggleMutation.mutate(!project.isFavorite)}
        disabled={toggleMutation.isPending}
      >
        {toggleMutation.isPending ? '...' : project.isFavorite ? '★' : '☆'}
      </button>
    </div>
  );
}
```

### Diagnostic Analysis: The Slow Feedback Problem

**Browser Behavior**:
1. User clicks star button
2. Button shows "..." for 200-500ms (network latency)
3. Star appears filled
4. Feels sluggish and unresponsive

**Network Tab**:
```
POST /api/projects/abc-123/favorite 200 OK (245ms)
GET /api/projects 200 OK (180ms)  ← Refetch after mutation
Total time: 425ms from click to UI update
```

**User expectation**: Instant feedback. The star should fill immediately when clicked.

**Let's parse this evidence**:

1. **What the user experiences**: Noticeable delay between clicking and seeing the result. Feels like the app is slow.

2. **What the Network tab reveals**: Two sequential requests—mutation then refetch. Total latency is the sum of both.

3. **Root cause identified**: We're waiting for the server to confirm the change before updating the UI.

4. **Why the current approach can't solve this**: Invalidation-based updates are inherently slow—they require a round trip to the server.

5. **What we need**: Update the UI immediately (optimistically) while the mutation is in flight, then reconcile with the server response.

## Iteration 7: Optimistic Updates

Optimistic updates mean updating the UI immediately, before the server responds. If the mutation fails, we roll back the change.

```tsx
// src/components/ProjectCard.tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

interface Project {
  id: string;
  name: string;
  isFavorite: boolean;
  // ... other fields
}

async function toggleFavorite(projectId: string, isFavorite: boolean): Promise<void> {
  const response = await fetch(`/api/projects/${projectId}/favorite`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ isFavorite }),
  });
  if (!response.ok) throw new Error('Failed to toggle favorite');
}

export function ProjectCard({ project }: { project: Project }) {
  const queryClient = useQueryClient();
  
  const toggleMutation = useMutation({
    mutationFn: (isFavorite: boolean) => toggleFavorite(project.id, isFavorite),
    
    // Before mutation starts
    onMutate: async (newIsFavorite) => {
      // Cancel any outgoing refetches (so they don't overwrite our optimistic update)
      await queryClient.cancelQueries({ queryKey: ['projects'] });
      
      // Snapshot the previous value
      const previousProjects = queryClient.getQueryData<Project[]>(['projects']);
      
      // Optimistically update the cache
      queryClient.setQueryData<Project[]>(['projects'], (old) => {
        if (!old) return old;
        return old.map((p) =>
          p.id === project.id ? { ...p, isFavorite: newIsFavorite } : p
        );
      });
      
      // Return context with the snapshot
      return { previousProjects };
    },
    
    // If mutation fails, roll back
    onError: (err, newIsFavorite, context) => {
      if (context?.previousProjects) {
        queryClient.setQueryData(['projects'], context.previousProjects);
      }
    },
    
    // Always refetch after error or success
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });
  
  return (
    <div className="project-card">
      <h3>{project.name}</h3>
      <button onClick={() => toggleMutation.mutate(!project.isFavorite)}>
        {project.isFavorite ? '★' : '☆'}
      </button>
    </div>
  );
}
```

### Verification: Instant Feedback

**Test sequence**:

1. Click star button
2. Star fills **instantly** (no delay)
3. Network request happens in background
4. If request succeeds, star stays filled
5. If request fails, star reverts to unfilled

**Browser Behavior**:
- Click → Instant visual feedback
- Feels responsive and snappy
- No loading state needed

**Network Tab**:
```
POST /api/projects/abc-123/favorite 200 OK (245ms)
(UI already updated—user doesn't wait for this)
GET /api/projects 200 OK (180ms)  ← Background refetch to confirm
```

**React Query DevTools**:
```
Query ['projects']:
  Status: success
  Data Age: 0s (just updated optimistically)
  Data: [{ id: 'abc-123', isFavorite: true, ... }]  ← Updated immediately
```

**What happened**:

1. User clicks button
2. `onMutate` runs **before** the network request:
   - Cancels any in-flight refetches
   - Saves current data as snapshot
   - Updates cache optimistically
3. Component re-renders with new data (star filled)
4. Network request happens in background
5. If success: `onSettled` refetches to confirm (usually matches optimistic update)
6. If error: `onError` restores snapshot (star reverts)

**Instant feedback. Optimistic by default. Automatic rollback on failure.**

### Understanding the Optimistic Update Flow

Let's break down each callback:

**onMutate** (runs before mutation):
- **Purpose**: Update cache optimistically
- **Returns**: Context object (snapshot for rollback)
- **When to use**: Always, for optimistic updates

**onError** (runs if mutation fails):
- **Purpose**: Roll back optimistic update
- **Receives**: Error, variables, context from onMutate
- **When to use**: Always, to restore previous state

**onSuccess** (runs if mutation succeeds):
- **Purpose**: Additional logic after success
- **Receives**: Response data, variables, context
- **When to use**: When you need to do something with the response

**onSettled** (runs after success or error):
- **Purpose**: Cleanup, refetch to confirm
- **Receives**: Data, error, variables, context
- **When to use**: Always, to refetch and ensure consistency

### Iteration 8: Optimistic Update with Server Response

Sometimes the server returns updated data. Use it to update the cache without refetching:

```tsx
// src/components/CreateProjectForm.tsx
interface CreateProjectResponse {
  project: Project;
  message: string;
}

async function createProject(data: CreateProjectData): Promise<CreateProjectResponse> {
  const response = await fetch('/api/projects', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to create project');
  return response.json();
}

export function CreateProjectForm() {
  const queryClient = useQueryClient();
  
  const mutation = useMutation({
    mutationFn: createProject,
    
    onMutate: async (newProject) => {
      await queryClient.cancelQueries({ queryKey: ['projects'] });
      const previousProjects = queryClient.getQueryData<Project[]>(['projects']);
      
      // Optimistically add the new project with a temporary ID
      queryClient.setQueryData<Project[]>(['projects'], (old) => {
        if (!old) return old;
        return [...old, { ...newProject, id: 'temp-' + Date.now() }];
      });
      
      return { previousProjects };
    },
    
    onSuccess: (response) => {
      // Replace temporary project with real one from server
      queryClient.setQueryData<Project[]>(['projects'], (old) => {
        if (!old) return old;
        return old.map((p) =>
          p.id.startsWith('temp-') ? response.project : p
        );
      });
    },
    
    onError: (err, newProject, context) => {
      if (context?.previousProjects) {
        queryClient.setQueryData(['projects'], context.previousProjects);
      }
    },
  });
  
  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      const formData = new FormData(e.currentTarget);
      mutation.mutate({
        name: formData.get('name') as string,
        description: formData.get('description') as string,
        status: 'active',
      });
    }}>
      <input name="name" placeholder="Project name" required />
      <textarea name="description" placeholder="Description" />
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create Project'}
      </button>
    </form>
  );
}
```

**What's different**:

1. **Temporary ID**: Optimistic project gets a temporary ID (`temp-123456`)
2. **onSuccess**: Replaces temporary project with real one from server
3. **No refetch needed**: Server response contains the complete project data

### Verification: Optimistic Create

**Test sequence**:

1. Submit form
2. New project appears in list **instantly** with temporary ID
3. Network request completes
4. Temporary project is replaced with real one (same position, real ID)
5. No visible flicker or re-render

**React Query DevTools**:
```
Before mutation:
Query ['projects']: [{ id: 'abc-123', ... }, { id: 'def-456', ... }]

After onMutate:
Query ['projects']: [{ id: 'abc-123', ... }, { id: 'def-456', ... }, { id: 'temp-1234567890', ... }]

After onSuccess:
Query ['projects']: [{ id: 'abc-123', ... }, { id: 'def-456', ... }, { id: 'ghi-789', ... }]
```

**Network Tab**:
```
POST /api/projects 201 Created (245ms)
(No GET request—server response contains the data we need)
```

**Instant feedback. No refetch. Seamless UX.**

### Common Failure Modes: Optimistic Updates

#### Symptom: Optimistic update doesn't appear

**Browser behavior**: Click button, nothing happens, then update appears after network request.

**Console pattern**:
```
(No errors, but optimistic update isn't visible)
```

**DevTools clues**:
- Cache is updated in DevTools
- Component doesn't re-render
- Query key mismatch

**Root cause**: Component is using a different query key than the mutation is updating.

**Solution**: Ensure mutation updates the same query key the component is using.

#### Symptom: Optimistic update flickers

**Browser behavior**: Update appears, then disappears briefly, then reappears.

**Console pattern**:
```
[React Query] Query ['projects'] refetched
```

**DevTools clues**:
- Query refetches immediately after optimistic update
- `cancelQueries` not called

**Root cause**: Forgot to cancel in-flight queries in `onMutate`.

**Solution**: Always call `await queryClient.cancelQueries()` in `onMutate`.

#### Symptom: Rollback doesn't work

**Browser behavior**: Mutation fails, but optimistic update stays (wrong data shown).

**Console pattern**:
```
Error: Failed to toggle favorite
(Optimistic update not rolled back)
```

**DevTools clues**:
- `onError` callback not defined
- Context not returned from `onMutate`

**Root cause**: Missing `onError` callback or not returning context from `onMutate`.

**Solution**: Always return context from `onMutate` and restore it in `onError`.

## When to Apply: Optimistic Update Decision Framework

### When to use optimistic updates

**Use optimistic updates when**:
- User action has predictable outcome (toggle, increment, simple update)
- Instant feedback improves UX significantly
- Failure is rare (< 1% of requests)
- Rollback is acceptable UX
- Examples: Like button, favorite toggle, simple status change

**Don't use optimistic updates when**:
- Outcome is unpredictable (complex validation, server-side computation)
- Failure is common (network issues, validation errors)
- Rollback would be confusing (e.g., payment processing)
- Server response contains critical data you don't have client-side
- Examples: Payment submission, complex form validation, file upload

### Optimistic update patterns

**Pattern 1: Simple toggle**
```tsx
onMutate: async (newValue) => {
  await queryClient.cancelQueries({ queryKey: ['item', id] });
  const previous = queryClient.getQueryData(['item', id]);
  queryClient.setQueryData(['item', id], { ...previous, field: newValue });
  return { previous };
},
onError: (err, newValue, context) => {
  queryClient.setQueryData(['item', id], context.previous);
}
```

**Pattern 2: List item update**
```tsx
onMutate: async (updates) => {
  await queryClient.cancelQueries({ queryKey: ['items'] });
  const previous = queryClient.getQueryData(['items']);
  queryClient.setQueryData(['items'], (old) =>
    old.map((item) => item.id === id ? { ...item, ...updates } : item)
  );
  return { previous };
}
```

**Pattern 3: List item creation**
```tsx
onMutate: async (newItem) => {
  await queryClient.cancelQueries({ queryKey: ['items'] });
  const previous = queryClient.getQueryData(['items']);
  queryClient.setQueryData(['items'], (old) => [...old, { ...newItem, id: 'temp-' + Date.now() }]);
  return { previous };
},
onSuccess: (response) => {
  queryClient.setQueryData(['items'], (old) =>
    old.map((item) => item.id.startsWith('temp-') ? response.item : item)
  );
}
```

**Pattern 4: List item deletion**
```tsx
onMutate: async (itemId) => {
  await queryClient.cancelQueries({ queryKey: ['items'] });
  const previous = queryClient.getQueryData(['items']);
  queryClient.setQueryData(['items'], (old) => old.filter((item) => item.id !== itemId));
  return { previous };
}
```

### Performance considerations

**Optimistic updates are fast because**:
- No network wait for UI update
- Cache update is synchronous
- Component re-renders immediately

**But watch out for**:
- Large lists: Updating 10,000 items in cache is slow
- Complex transformations: Keep optimistic logic simple
- Multiple related queries: Update all affected queries or none

**Optimization**: If updating a large list, consider invalidating instead of optimistic update for better performance.

## Replacing your Redux boilerplate

## The Journey: From Redux to React Query

Many React applications use Redux for all state management, including server data. Let's see how React Query eliminates the need for Redux in most cases.

### The Redux Approach (What We're Replacing)

Here's a typical Redux setup for managing projects:

```typescript
// src/store/projectsSlice.ts (Redux Toolkit)
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

interface ProjectsState {
  items: Project[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const initialState: ProjectsState = {
  items: [],
  status: 'idle',
  error: null,
};

// Async thunk for fetching projects
export const fetchProjects = createAsyncThunk(
  'projects/fetchProjects',
  async () => {
    const response = await fetch('/api/projects');
    return response.json();
  }
);

// Async thunk for creating project
export const createProject = createAsyncThunk(
  'projects/createProject',
  async (data: CreateProjectData) => {
    const response = await fetch('/api/projects', {
      method: 'POST',
      body: JSON.stringify(data),
    });
    return response.json();
  }
);

const projectsSlice = createSlice({
  name: 'projects',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchProjects.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchProjects.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(fetchProjects.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message ?? 'Failed to fetch';
      })
      .addCase(createProject.fulfilled, (state, action) => {
        state.items.push(action.payload);
      });
  },
});

export default projectsSlice.reducer;
```

```tsx
// src/components/ProjectDashboard.tsx (Redux version)
import { useEffect } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { fetchProjects } from '../store/projectsSlice';

export function ProjectDashboard() {
  const dispatch = useDispatch();
  const { items: projects, status, error } = useSelector(
    (state: RootState) => state.projects
  );
  
  useEffect(() => {
    if (status === 'idle') {
      dispatch(fetchProjects());
    }
  }, [status, dispatch]);
  
  if (status === 'loading') return <div>Loading...</div>;
  if (status === 'failed') return <div>Error: {error}</div>;
  
  return (
    <div className="dashboard">
      <h1>Projects ({projects.length})</h1>
      <div className="project-grid">
        {projects.map(project => (
          <ProjectCard key={project.id} project={project} />
        ))}
      </div>
    </div>
  );
}
```

**Problems with this approach**:

1. **Boilerplate**: 50+ lines of code for a simple fetch
2. **Manual cache management**: No automatic refetching, no staleness detection
3. **No deduplication**: Multiple components fetching the same data make duplicate requests
4. **Manual invalidation**: After mutations, you manually update the Redux store
5. **Global state pollution**: Server data mixed with client state
6. **No background refetching**: Data becomes stale, no automatic updates
7. **Complex error handling**: Need to handle errors in multiple places

### The React Query Approach (What We're Moving To)

Here's the same functionality with React Query:

```tsx
// src/api/projects.ts (Pure API functions)
export async function fetchProjects(): Promise<Project[]> {
  const response = await fetch('/api/projects');
  if (!response.ok) throw new Error('Failed to fetch projects');
  return response.json();
}

export async function createProject(data: CreateProjectData): Promise<Project> {
  const response = await fetch('/api/projects', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to create project');
  return response.json();
}
```

```tsx
// src/components/ProjectDashboard.tsx (React Query version)
import { useQuery } from '@tanstack/react-query';
import { fetchProjects } from '../api/projects';

export function ProjectDashboard() {
  const { data: projects, isLoading, error } = useQuery({
    queryKey: ['projects'],
    queryFn: fetchProjects,
  });
  
  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  return (
    <div className="dashboard">
      <h1>Projects ({projects?.length ?? 0})</h1>
      <div className="project-grid">
        {projects?.map(project => (
          <ProjectCard key={project.id} project={project} />
        ))}
      </div>
    </div>
  );
}
```

**What we gained**:

1. **90% less code**: 10 lines vs. 50+ lines
2. **Automatic caching**: No manual cache management
3. **Automatic deduplication**: Multiple components share the same query
4. **Automatic refetching**: Stale data refetches on window focus, reconnect, etc.
5. **Automatic invalidation**: Mutations invalidate related queries
6. **Background updates**: Data stays fresh without user intervention
7. **Built-in error handling**: Retry logic, error states, all handled

### Iteration 9: Complete Migration Example

Let's migrate a complete Redux feature to React Query:

```typescript
// BEFORE: Redux approach
// src/store/projectsSlice.ts (100+ lines)
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

export const fetchProjects = createAsyncThunk('projects/fetch', async () => {
  const response = await fetch('/api/projects');
  return response.json();
});

export const fetchProjectDetails = createAsyncThunk(
  'projects/fetchDetails',
  async (id: string) => {
    const response = await fetch(`/api/projects/${id}`);
    return response.json();
  }
);

export const createProject = createAsyncThunk(
  'projects/create',
  async (data: CreateProjectData) => {
    const response = await fetch('/api/projects', {
      method: 'POST',
      body: JSON.stringify(data),
    });
    return response.json();
  }
);

export const updateProject = createAsyncThunk(
  'projects/update',
  async ({ id, data }: { id: string; data: Partial<Project> }) => {
    const response = await fetch(`/api/projects/${id}`, {
      method: 'PATCH',
      body: JSON.stringify(data),
    });
    return response.json();
  }
);

export const deleteProject = createAsyncThunk(
  'projects/delete',
  async (id: string) => {
    await fetch(`/api/projects/${id}`, { method: 'DELETE' });
    return id;
  }
);

const projectsSlice = createSlice({
  name: 'projects',
  initialState: {
    list: [],
    details: {},
    listStatus: 'idle',
    detailsStatus: {},
    error: null,
  },
  reducers: {},
  extraReducers: (builder) => {
    // 50+ lines of reducer logic for each action...
  },
});
```

```typescript
// AFTER: React Query approach
// src/api/projects.ts (30 lines)
export async function fetchProjects(): Promise<Project[]> {
  const response = await fetch('/api/projects');
  if (!response.ok) throw new Error('Failed to fetch projects');
  return response.json();
}

export async function fetchProjectDetails(id: string): Promise<ProjectDetails> {
  const response = await fetch(`/api/projects/${id}`);
  if (!response.ok) throw new Error('Failed to fetch project details');
  return response.json();
}

export async function createProject(data: CreateProjectData): Promise<Project> {
  const response = await fetch('/api/projects', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to create project');
  return response.json();
}

export async function updateProject(
  id: string,
  data: Partial<Project>
): Promise<Project> {
  const response = await fetch(`/api/projects/${id}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to update project');
  return response.json();
}

export async function deleteProject(id: string): Promise<void> {
  const response = await fetch(`/api/projects/${id}`, { method: 'DELETE' });
  if (!response.ok) throw new Error('Failed to delete project');
}
```

```tsx
// src/hooks/useProjects.ts (Custom hooks for reusability)
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import * as api from '../api/projects';

export function useProjects() {
  return useQuery({
    queryKey: ['projects'],
    queryFn: api.fetchProjects,
  });
}

export function useProjectDetails(id: string) {
  return useQuery({
    queryKey: ['project', id],
    queryFn: () => api.fetchProjectDetails(id),
    enabled: !!id,
  });
}

export function useCreateProject() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: api.createProject,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });
}

export function useUpdateProject() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<Project> }) =>
      api.updateProject(id, data),
    onSuccess: (_, { id }) => {
      queryClient.invalidateQueries({ queryKey: ['projects'] });
      queryClient.invalidateQueries({ queryKey: ['project', id] });
    },
  });
}

export function useDeleteProject() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: api.deleteProject,
    onSuccess: (_, id) => {
      queryClient.removeQueries({ queryKey: ['project', id] });
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });
}
```

```tsx
// src/components/ProjectDashboard.tsx (Using custom hooks)
import { useProjects, useCreateProject } from '../hooks/useProjects';

export function ProjectDashboard() {
  const { data: projects, isLoading, error } = useProjects();
  const createMutation = useCreateProject();
  
  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  return (
    <div className="dashboard">
      <h1>Projects ({projects?.length ?? 0})</h1>
      <CreateProjectForm mutation={createMutation} />
      <div className="project-grid">
        {projects?.map(project => (
          <ProjectCard key={project.id} project={project} />
        ))}
      </div>
    </div>
  );
}
```

**Code comparison**:

| Aspect | Redux | React Query |
|--------|-------|-------------|
| Lines of code | 150+ | 50 |
| Boilerplate | High | Minimal |
| Cache management | Manual | Automatic |
| Refetching | Manual | Automatic |
| Deduplication | Manual | Automatic |
| Invalidation | Manual | Automatic |
| Loading states | Manual | Built-in |
| Error handling | Manual | Built-in |
| Optimistic updates | Complex | Simple |

### What About Client State?

React Query handles **server state**. You still need something for **client state** (UI state, form state, user preferences). But now you can use simpler tools:

**For local component state**: `useState`, `useReducer`

**For global client state**: Zustand (much simpler than Redux)

```tsx
// src/store/uiStore.ts (Zustand for client state)
import { create } from 'zustand';

interface UIStore {
  sidebarOpen: boolean;
  theme: 'light' | 'dark';
  toggleSidebar: () => void;
  setTheme: (theme: 'light' | 'dark') => void;
}

export const useUIStore = create<UIStore>((set) => ({
  sidebarOpen: true,
  theme: 'light',
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
  setTheme: (theme) => set({ theme }),
}));
```

**Architecture**:

- **React Query**: All server data (API calls, database queries)
- **Zustand**: Global client state (UI state, user preferences)
- **useState/useReducer**: Local component state (form inputs, modal open/closed)

This separation is cleaner and more maintainable than putting everything in Redux.

## The Complete Journey: From Redux to React Query

### Migration Strategy

**Phase 1: Set up React Query**
1. Install `@tanstack/react-query`
2. Add `QueryClientProvider` to your app
3. Install React Query DevTools

**Phase 2: Identify server state**
1. List all Redux slices that manage server data
2. Identify API calls in Redux thunks
3. Separate server state from client state

**Phase 3: Migrate one feature at a time**
1. Extract API functions from Redux thunks
2. Create custom hooks with `useQuery` and `useMutation`
3. Replace Redux `useSelector` with React Query hooks
4. Remove Redux slice once feature is fully migrated

**Phase 4: Clean up**
1. Remove unused Redux code
2. Simplify remaining Redux store (only client state)
3. Consider replacing Redux with Zustand for client state

### Decision Framework: Redux vs. React Query vs. Zustand

**Use React Query when**:
- Data comes from an API
- Data can change on the server
- Multiple components need the same data
- You need caching, refetching, or background updates
- Examples: User profiles, project lists, search results

**Use Zustand when**:
- Data is client-only (doesn't exist on server)
- Multiple components need to share state
- State changes frequently
- Examples: UI state, user preferences, form wizards

**Use useState/useReducer when**:
- State is local to one component
- State doesn't need to be shared
- Simple state logic
- Examples: Form inputs, modal open/closed, accordion expanded

**Don't use Redux when**:
- You're managing server state (use React Query)
- You're managing simple client state (use Zustand)
- You're managing local state (use useState)

**Still use Redux when**:
- You have a massive existing Redux codebase
- Your team is deeply invested in Redux patterns
- You need Redux DevTools time-travel debugging
- You're managing complex client state with many interdependencies

But even then, consider migrating server state to React Query and keeping Redux only for client state.

## Final Implementation: Production-Ready Project Dashboard

Here's our complete Project Dashboard with all React Query features:

```typescript
// src/api/projects.ts
export interface Project {
  id: string;
  name: string;
  description: string;
  status: 'active' | 'completed' | 'archived';
  isFavorite: boolean;
  taskCount: number;
  teamSize: number;
  createdAt: string;
  updatedAt: string;
}

export interface ProjectDetails extends Project {
  tasks: Task[];
  team: TeamMember[];
}

export interface CreateProjectData {
  name: string;
  description: string;
  status: 'active' | 'completed' | 'archived';
}

export async function fetchProjects(): Promise<Project[]> {
  const response = await fetch('/api/projects');
  if (!response.ok) throw new Error('Failed to fetch projects');
  return response.json();
}

export async function fetchProjectDetails(id: string): Promise<ProjectDetails> {
  const response = await fetch(`/api/projects/${id}`);
  if (!response.ok) throw new Error('Failed to fetch project details');
  return response.json();
}

export async function createProject(data: CreateProjectData): Promise<Project> {
  const response = await fetch('/api/projects', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to create project');
  return response.json();
}

export async function updateProject(
  id: string,
  data: Partial<Project>
): Promise<Project> {
  const response = await fetch(`/api/projects/${id}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to update project');
  return response.json();
}

export async function deleteProject(id: string): Promise<void> {
  const response = await fetch(`/api/projects/${id}`, { method: 'DELETE' });
  if (!response.ok) throw new Error('Failed to delete project');
}

export async function toggleFavorite(
  id: string,
  isFavorite: boolean
): Promise<void> {
  const response = await fetch(`/api/projects/${id}/favorite`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ isFavorite }),
  });
  if (!response.ok) throw new Error('Failed to toggle favorite');
}

export async function searchProjects(query: string): Promise<Project[]> {
  const response = await fetch(`/api/projects/search?q=${encodeURIComponent(query)}`);
  if (!response.ok) throw new Error('Search failed');
  return response.json();
}
```

```typescript
// src/hooks/useProjects.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import * as api from '../api/projects';

export function useProjects() {
  return useQuery({
    queryKey: ['projects'],
    queryFn: api.fetchProjects,
    staleTime: 1000 * 60 * 5, // 5 minutes
  });
}

export function useProjectDetails(id: string) {
  return useQuery({
    queryKey: ['project', id],
    queryFn: () => api.fetchProjectDetails(id),
    enabled: !!id,
    staleTime: 1000 * 60 * 5,
  });
}

export function useProjectSearch(query: string) {
  return useQuery({
    queryKey: ['projects', 'search', query],
    queryFn: () => api.searchProjects(query),
    enabled: query.length >= 3,
    staleTime: 1000 * 60, // 1 minute
  });
}

export function useCreateProject() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: api.createProject,
    onMutate: async (newProject) => {
      await queryClient.cancelQueries({ queryKey: ['projects'] });
      const previous = queryClient.getQueryData(['projects']);
      
      queryClient.setQueryData(['projects'], (old: api.Project[] | undefined) => {
        if (!old) return old;
        return [...old, { ...newProject, id: 'temp-' + Date.now() }];
      });
      
      return { previous };
    },
    onSuccess: (response) => {
      queryClient.setQueryData(['projects'], (old: api.Project[] | undefined) => {
        if (!old) return old;
        return old.map((p) => (p.id.startsWith('temp-') ? response : p));
      });
    },
    onError: (err, newProject, context) => {
      if (context?.previous) {
        queryClient.setQueryData(['projects'], context.previous);
      }
    },
  });
}

export function useUpdateProject() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<api.Project> }) =>
      api.updateProject(id, data),
    onSuccess: (_, { id }) => {
      queryClient.invalidateQueries({ queryKey: ['projects'] });
      queryClient.invalidateQueries({ queryKey: ['project', id] });
    },
  });
}

export function useDeleteProject() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: api.deleteProject,
    onSuccess: (_, id) => {
      queryClient.removeQueries({ queryKey: ['project', id] });
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });
}

export function useToggleFavorite() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ id, isFavorite }: { id: string; isFavorite: boolean }) =>
      api.toggleFavorite(id, isFavorite),
    onMutate: async ({ id, isFavorite }) => {
      await queryClient.cancelQueries({ queryKey: ['projects'] });
      const previous = queryClient.getQueryData(['projects']);
      
      queryClient.setQueryData(['projects'], (old: api.Project[] | undefined) => {
        if (!old) return old;
        return old.map((p) => (p.id === id ? { ...p, isFavorite } : p));
      });
      
      return { previous };
    },
    onError: (err, variables, context) => {
      if (context?.previous) {
        queryClient.setQueryData(['projects'], context.previous);
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['projects'] });
    },
  });
}
```

```tsx
// src/components/ProjectDashboard.tsx
import { useState } from 'react';
import { useProjects, useProjectSearch } from '../hooks/useProjects';
import { ProjectCard } from './ProjectCard';
import { CreateProjectForm } from './CreateProjectForm';
import { ProjectDetailsModal } from './ProjectDetailsModal';

export function ProjectDashboard() {
  const [searchQuery, setSearchQuery] = useState('');
  const [selectedProjectId, setSelectedProjectId] = useState<string | null>(null);
  
  const { data: allProjects, isLoading, isFetching, error } = useProjects();
  const { data: searchResults } = useProjectSearch(searchQuery);
  
  const projects = searchQuery.length >= 3 ? searchResults : allProjects;
  
  if (isLoading) {
    return <div className="loading">Loading projects...</div>;
  }
  
  if (error) {
    return <div className="error">Error: {error.message}</div>;
  }
  
  return (
    <div className="dashboard">
      <header>
        <h1>Projects ({projects?.length ?? 0})</h1>
        {isFetching && <span className="refetch-indicator">Updating...</span>}
      </header>
      
      <div className="controls">
        <input
          type="search"
          value={searchQuery}
          onChange={(e) => setSearchQuery(e.target.value)}
          placeholder="Search projects..."
          className="search-input"
        />
        <CreateProjectForm />
      </div>
      
      <div className="project-grid">
        {projects?.map((project) => (
          <ProjectCard
            key={project.id}
            project={project}
            onClick={() => setSelectedProjectId(project.id)}
          />
        ))}
      </div>
      
      {selectedProjectId && (
        <ProjectDetailsModal
          projectId={selectedProjectId}
          onClose={() => setSelectedProjectId(null)}
        />
      )}
    </div>
  );
}
```

```tsx
// src/components/ProjectCard.tsx
import { useToggleFavorite, useDeleteProject } from '../hooks/useProjects';
import type { Project } from '../api/projects';

interface ProjectCardProps {
  project: Project;
  onClick: () => void;
}

export function ProjectCard({ project, onClick }: ProjectCardProps) {
  const toggleFavorite = useToggleFavorite();
  const deleteProject = useDeleteProject();
  
  const handleToggleFavorite = (e: React.MouseEvent) => {
    e.stopPropagation();
    toggleFavorite.mutate({ id: project.id, isFavorite: !project.isFavorite });
  };
  
  const handleDelete = (e: React.MouseEvent) => {
    e.stopPropagation();
    if (confirm(`Delete project "${project.name}"?`)) {
      deleteProject.mutate(project.id);
    }
  };
  
  return (
    <div className="project-card" onClick={onClick}>
      <div className="card-header">
        <h3>{project.name}</h3>
        <button
          onClick={handleToggleFavorite}
          className="favorite-button"
          aria-label={project.isFavorite ? 'Remove from favorites' : 'Add to favorites'}
        >
          {project.isFavorite ? '★' : '☆'}
        </button>
      </div>
      
      <p className="description">{project.description}</p>
      
      <div className="card-meta">
        <span className={`status status-${project.status}`}>
          {project.status}
        </span>
        <span>{project.taskCount} tasks</span>
        <span>{project.teamSize} members</span>
      </div>
      
      <div className="card-actions">
        <button
          onClick={handleDelete}
          disabled={deleteProject.isPending}
          className="delete-button"
        >
          {deleteProject.isPending ? 'Deleting...' : 'Delete'}
        </button>
      </div>
    </div>
  );
}
```

```tsx
// src/components/CreateProjectForm.tsx
import { useState } from 'react';
import { useCreateProject } from '../hooks/useProjects';

export function CreateProjectForm() {
  const [isOpen, setIsOpen] = useState(false);
  const [formData, setFormData] = useState({
    name: '',
    description: '',
    status: 'active' as const,
  });
  
  const createProject = useCreateProject();
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    createProject.mutate(formData, {
      onSuccess: () => {
        setFormData({ name: '', description: '', status: 'active' });
        setIsOpen(false);
      },
    });
  };
  
  if (!isOpen) {
    return (
      <button onClick={() => setIsOpen(true)} className="create-button">
        + New Project
      </button>
    );
  }
  
  return (
    <form onSubmit={handleSubmit} className="create-form">
      <input
        value={formData.name}
        onChange={(e) => setFormData({ ...formData, name: e.target.value })}
        placeholder="Project name"
        required
        autoFocus
      />
      <textarea
        value={formData.description}
        onChange={(e) => setFormData({ ...formData, description: e.target.value })}
        placeholder="Description"
        rows={3}
      />
      <div className="form-actions">
        <button type="submit" disabled={createProject.isPending}>
          {createProject.isPending ? 'Creating...' : 'Create'}
        </button>
        <button type="button" onClick={() => setIsOpen(false)}>
          Cancel
        </button>
      </div>
      {createProject.isError && (
        <div className="error">{createProject.error.message}</div>
      )}
    </form>
  );
}
```
