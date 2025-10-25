# Chapter 25: Type-Safe API Integration

## Typing REST API Responses

## Learning Objective

Define TypeScript types for expected JSON responses from a REST API and use type assertions to safely handle the fetched data.

## Why This Matters

The boundary between your application and an external API is the most common source of runtime errors. Data arrives as untyped JSON, and assuming its shape can lead to crashes like `TypeError: Cannot read properties of undefined`. By defining the shape of API data upfront, you can catch these mismatches during development.

## Discovery Phase

Let's start with a standard `fetch` call in a component. We want to fetch a user's data.

```jsx
import React, { useEffect, useState } from "react";

function UserProfile() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch("https://jsonplaceholder.typicode.com/users/1")
      .then((res) => res.json())
      .then((jsonData) => {
        // What is the type of `jsonData` here? It's `any`.
        // This is dangerous! The compiler can't help us.
        setData(jsonData);
      });
  }, []);

  // We have to guess the shape of the data and use optional chaining for safety.
  return <div>Welcome, {data?.name}</div>;
}
```

The problem is that `res.json()` returns a promise that resolves to a value of type `any`. TypeScript has no way of knowing the shape of the data coming from the network.

The first step to fixing this is to define the shape we expect.

```jsx
import React, { useEffect, useState } from 'react';

// Step 1: Define the type for the API response.
interface User {
  id: number;
  name: string;
  username: string;
  email: string;
  address: {
    street: string;
    city: string;
  };
}

function TypedUserProfile() {
  // We can use our new type to type our state.
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    const fetchUser = async () => {
      const response = await fetch('https://jsonplaceholder.typicode.com/users/1');
      // Step 2: Use a type assertion (`as`) to tell TypeScript the shape.
      const userData = (await response.json()) as User;
      setUser(userData);
    };
    fetchUser();
  }, []);

  // Now, TypeScript knows the shape of `user`!
  // We get autocompletion for `user.name`, `user.address.city`, etc.
  return <div>Welcome, {user?.name}</div>;
}
```

By defining the `User` interface and using the type assertion `as User`, we've created a contract. We are telling TypeScript, "I promise that the data I get from this API will have this shape." Now, our component's state is typed, and we get all the benefits of static analysis.

## Deep Dive

### The Nature of Type Assertions

A type assertion (`as User`) is a way to override TypeScript's inferred type. It's important to understand that **this does not perform any runtime checking**. It is purely a message to the compiler.

- **You are taking responsibility**: You are telling the compiler that you know more about the type of a value than it does.
- **It can be unsafe**: If the API changes and `name` becomes `fullName`, your code will still compile, but `user.name` will be `undefined` at runtime.

This is a trade-off. A type assertion gives you immediate development-time safety and autocompletion at the cost of trusting the external API. In section 25.7, we'll see how to verify this trust at runtime using a library like Zod.

### Common Confusion: "Why can't TypeScript just know the API's type?"

**You might think**: "If I can visit the API URL in my browser and see the JSON, why can't the compiler do the same?"

**Actually**: TypeScript's type checking happens at _build time_, before your code is ever run. It analyzes your static code; it does not make network requests. The data from an API only exists at _runtime_, when a user is actually using your application. There's a fundamental wall between the build-time world of types and the runtime world of data.

**How to remember**: TypeScript types are erased when your code is compiled to JavaScript. The code running in the browser has no concept of your `User` interface. The type assertion is your way of bridging this gap.

### Production Perspective

- **API Contracts**: In professional teams, the "single source of truth" for API shapes is often a formal specification like an OpenAPI (formerly Swagger) document. There are tools that can automatically generate TypeScript interfaces from these specifications, which eliminates manual typing and ensures the frontend and backend are always in sync.
- **Error Handling**: Always couple your fetching logic with robust error handling. What happens if the `fetch` fails or the server returns a 500 error? Your type definition describes the _success_ case, but your code must handle the failure cases as well.

## Type-Safe Fetch Wrappers

## Learning Objective

Create a generic, reusable fetch wrapper function that centralizes API logic, handles JSON parsing and errors, and returns strongly-typed data.

## Why This Matters

Repeating `fetch` calls, `.json()` parsing, and `try/catch` blocks in every component that needs data is a violation of the DRY (Don't Repeat Yourself) principle. A centralized fetch wrapper makes your data-fetching code cleaner, more consistent, and easier to maintain. Making it generic allows it to be used for any API endpoint and any data type.

## Discovery Phase

Let's take the fetching logic from the previous section and extract it into a reusable function.

```javascript
// FileName: utils/api.ts

// Step 1: Make the function generic with `<T>`.
// This `T` is a placeholder for whatever data type we expect to get back.
export async function apiFetch<T>(url: string): Promise<T> {
  try {
    const response = await fetch(url);

    // Step 2: Handle non-successful responses.
    if (!response.ok) {
      throw new Error(`API error: ${response.status} ${response.statusText}`);
    }

    // Step 3: Parse the JSON and cast it to the generic type `T`.
    const data = (await response.json()) as T;
    return data;
  } catch (error) {
    console.error('Fetch error:', error);
    // Re-throw the error so the calling component can handle it.
    throw error;
  }
}
```

This generic wrapper encapsulates all the boilerplate. Now, our component becomes incredibly simple.

```jsx
// FileName: components/TypedUserProfile.tsx
import React, { useEffect, useState } from 'react';
import { apiFetch } from '../utils/api';

// The User interface is still needed to describe the data shape.
interface User {
  id: number;
  name: string;
}

function TypedUserProfile() {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    // Step 4: Use the wrapper and provide the generic type argument.
    apiFetch<User>('https://jsonplaceholder.typicode.com/users/1')
      .then(setUser)
      .catch(setError);
  }, []);

  if (error) return <div>Error: {error.message}</div>;
  if (!user) return <div>Loading...</div>;

  // `user` is guaranteed to be of type `User` here.
  return <div>Welcome, {user.name}</div>;
}
```

The component's responsibility is now much clearer: it asks for data of type `User` and handles the three possible states (loading, success, error). All the implementation details of how to fetch, parse, and handle HTTP errors are hidden away in the `apiFetch` wrapper.

## Deep Dive

### Expanding the Wrapper for `POST` Requests

We can easily extend our wrapper to handle data mutations with methods like `POST`.

```javascript
// FileName: utils/api.ts (extended)

// Add an optional `options` parameter.
export async function apiFetch<T>(url: string, options?: RequestInit): Promise<T> {
  // ... same implementation as before
  const response = await fetch(url, options);
  // ...
  const data = (await response.json()) as T;
  return data;
}

// --- USAGE IN A COMPONENT ---
type NewPost = { title: string; body: string; };
type CreatedPost = NewPost & { id: number; };

async function createPost(newPost: NewPost) {
  const createdPost = await apiFetch<CreatedPost>(
    'https://jsonplaceholder.typicode.com/posts',
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(newPost),
    }
  );
  console.log('Created post:', createdPost);
}
```

By accepting the standard `RequestInit` object, our wrapper can handle any valid `fetch` option, making it a versatile tool for all API interactions.

### Production Perspective

- **Centralized API Client**: In a large application, this simple wrapper often evolves into a more comprehensive API client class or module. This client might handle things like:
  - Automatically adding authentication headers (`Authorization: Bearer ...`).
  - Handling token refresh logic.
  - Using a base URL so you only need to provide endpoints (`api.get('/users/1')`).
  - Consistent logging and error reporting.
- **Don't Reinvent the Wheel (Usually)**: While building a simple wrapper is a great learning exercise and sufficient for many projects, libraries like `axios` or `ky` provide more advanced features out of the box. However, the core concept of a generic function that returns a typed promise remains the same. The real leap forward comes from libraries like React Query, which manage the state _around_ your fetch calls.

## React Query with TypeScript

## Learning Objective

Use TanStack Query (React Query) with TypeScript to manage server state, leveraging its generic types for `useQuery` and `useMutation` to create a robust and declarative data-fetching layer.

## Why This Matters

Fetching data is only half the battle. You also need to manage loading states, errors, caching, re-fetching on window focus, and optimistic updates. React Query handles all of this for you. Its first-class TypeScript support means you get all this power with full type safety, making it the industry standard for managing server state in React applications.

## Discovery Phase

Let's refactor our user profile component to use `useQuery`.

```jsx
import { useQuery } from '@tanstack/react-query';
import React from 'react';

// Define the data shape and the fetcher function.
interface User {
  id: number;
  name: string;
}

const fetchUser = async (id: number): Promise<User> => {
  const res = await fetch(`https://jsonplaceholder.typicode.com/users/${id}`);
  if (!res.ok) throw new Error('Network response was not ok');
  return res.json();
};

function UserProfile({ userId }: { userId: number }) {
  // Step 1: Use the `useQuery` hook.
  const { data, isLoading, isError, error } = useQuery<User, Error>({
    // A unique key for this query.
    queryKey: ['user', userId],
    // The function that fetches the data.
    queryFn: () => fetchUser(userId),
  });

  // Step 2: React Query manages the state for us.
  if (isLoading) return <div>Loading...</div>;
  if (isError) return <div>Error: {error.message}</div>;

  // Step 3: The `data` is fully typed.
  // TypeScript knows `data` is of type `User` at this point.
  return <div>Welcome, {data.name}</div>;
}
```

This is a huge improvement over `useState` and `useEffect`.

1.  **Declarative**: We declare what data we need (`queryKey`) and how to get it (`queryFn`). React Query handles the rest.
2.  **Typed State**: The hook returns a fully typed object. `isLoading` is a boolean, `error` is an `Error`, and `data` is typed as `User`.
3.  **Generics**: The `useQuery<User, Error>` signature tells the hook two things: the success data will be of type `User`, and the error will be of type `Error`.

## Deep Dive

### Type Inference from `queryFn`

Often, you don't even need to provide the generic argument if your `queryFn` is correctly typed.

```jsx
// `fetchUser` already has a return type of `Promise<User>`.
const fetchUser = async (id: number): Promise<User> => { /* ... */ };

function UserProfile({ userId }: { userId: number }) {
  // No generic needed here!
  // React Query infers the `data` type from the return type of `queryFn`.
  const { data, ... } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });
  // `data` is still correctly inferred as `User | undefined`.
}
```

This is the preferred modern pattern: write well-typed fetcher functions, and let React Query's inference do the work.

### Typing Mutations with `useMutation`

`useMutation` is used for creating, updating, or deleting data. It also has excellent generic support.

```jsx
import { useMutation } from '@tanstack/react-query';

type NewPost = { title: string; body: string; };
type CreatedPost = NewPost & { id: number; };

const createPost = async (newPost: NewPost): Promise<CreatedPost> => {
  const res = await fetch('https://jsonplaceholder.typicode.com/posts', {
    method: 'POST',
    body: JSON.stringify(newPost),
    headers: { 'Content-Type': 'application/json' },
  });
  return res.json();
};

function CreatePostForm() {
  // The generic signature is: `useMutation<TData, TError, TVariables>`
  const mutation = useMutation<CreatedPost, Error, NewPost>({
    mutationFn: createPost,
    onSuccess: (data) => {
      // `data` is typed as `CreatedPost`.
      alert(`Post created with ID: ${data.id}`);
    },
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    // `mutation.mutate` expects an argument of type `NewPost`.
    mutation.mutate({ title: 'My New Post', body: '...' });
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* ... form fields ... */}
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Saving...' : 'Create Post'}
      </button>
    </form>
  );
}
```

The generics for `useMutation` define the contract:

- `TData` (`CreatedPost`): The type of data returned on success.
- `TError` (`Error`): The type of the error on failure.
- `TVariables` (`NewPost`): The type of the variables passed to the `mutate` function.

### Production Perspective

- **Server State vs. Client State**: React Query makes a clear distinction. Use React Query for data that lives on your server. Use `useState` or `useReducer` for ephemeral UI state that only exists on the client (like whether a modal is open).
- **Caching is the Killer Feature**: The primary benefit of React Query is its intelligent caching. It automatically handles re-fetching in the background to keep your UI in sync with the server, reducing the need for manual data fetching logic.
- **DevTools**: The React Query DevTools are an indispensable tool for debugging, allowing you to inspect the query cache, see the status of each query, and manually trigger actions.

## GraphQL Code Generation

## Learning Objective

Use GraphQL Code Generator to automatically create TypeScript types and typed hooks from a GraphQL schema and query operations, eliminating manual type definitions.

## Why This Matters

The single biggest advantage of GraphQL is its strongly typed schema. It's the ultimate source of truth for your API's shape. Manually writing TypeScript types to match this schema is redundant and error-prone. Code generation automates this process, guaranteeing that your frontend types are always a perfect mirror of your backend API.

## Discovery Phase

Let's start with a simple GraphQL query.

```graphql
# FileName: queries/user.graphql

query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
  }
}
```

Without code generation, we would have to manually create the TypeScript types for the query result and variables.

```typescript
// Manual, error-prone, and not maintainable
interface GetUserQueryResult {
  user: {
    id: string;
    name: string;
    email: string;
  };
}
interface GetUserQueryVariables {
  id: string;
}
```

This is a recipe for disaster. If the backend renames `email` to `emailAddress`, our types become incorrect, and we won't know until we get a runtime error.

**The Code Generation Workflow:**

1.  **Configuration**: You create a `codegen.yml` file to configure the generator.

```yaml
# FileName: codegen.yml (simplified)

# The URL of your GraphQL API endpoint or a local schema file
schema: "https://my-api.com/graphql"

# Globs pointing to your files containing GraphQL queries/mutations
documents: "src/**/*.graphql"

# What to generate
generates:
  # Where to put the output file
  src/gql/generated.ts:
    # A list of plugins to use
    plugins:
      - "typescript" # Generates base types for your schema
      - "typescript-operations" # Generates types for your queries/mutations
      - "typescript-react-apollo" # Generates typed hooks for React Apollo
```

2.  **Run the Generator**: You run a command in your terminal, like `graphql-codegen`.
3.  **Use the Generated Code**: The generator creates a file with all the types you need.

```typescript
// FileName: src/gql/generated.ts (This file is AUTO-GENERATED)

// Base types from your schema
export type User = {
  __typename?: "User";
  id: Scalars["ID"];
  name: Scalars["String"];
  email: Scalars["String"];
};

// Types for our specific query
export type GetUserQueryVariables = { id: Scalars["ID"] };
export type GetUserQuery = {
  __typename?: "Query";
  user?: {
    __typename?: "User";
    id: string;
    name: string;
    email: string;
  } | null;
};

// A fully typed, ready-to-use hook!
export function useGetUserQuery(
  baseOptions: Apollo.QueryHookOptions<GetUserQuery, GetUserQueryVariables>,
) {
  // ... implementation generated by the plugin
}
```

Now, in our component, we don't write any types. We just import and use the generated hook.

```jsx
import React from 'react';
// Import the auto-generated hook
import { useGetUserQuery } from '../gql/generated';

function UserProfile({ userId }: { userId: string }) {
  // Use the generated hook. No manual typing needed.
  const { data, loading, error } = useGetUserQuery({
    variables: { id: userId },
  });

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error!</p>;

  // `data` is fully typed as `GetUserQuery | undefined`.
  // We get autocompletion for `data.user.name`.
  return <div>Welcome, {data?.user?.name}</div>;
}
```

This is the gold standard for type-safe GraphQL. The types are guaranteed to be correct because they are generated directly from the API's schema.

### Production Perspective

- **The Single Source of Truth**: This workflow makes the GraphQL schema the undisputed source of truth. Any change to the schema or a query will be immediately reflected in the generated types after re-running the generator, often leading to compile-time errors on the frontend if there's a breaking change. This is a huge benefit.
- **CI/CD Integration**: It's a common practice to run the code generator as part of your Continuous Integration (CI) pipeline. This ensures that your application won't even build if the frontend's queries are incompatible with the latest version of the backend schema.

## Apollo Client Type Safety

## Learning Objective

Integrate generated GraphQL types with Apollo Client's `useQuery` and `useMutation` hooks, and understand how code generation can create fully-typed hooks automatically.

## Why This Matters

Apollo Client is a feature-rich GraphQL client that provides caching, local state management, and more. While GraphQL Code Generator can create fully-typed hooks for you, it's also important to understand how to use Apollo's base hooks with the generated types, as you may need to do this for more complex or dynamic queries.

## Discovery Phase

Let's assume we've used GraphQL Code Generator, but only with the `typescript` and `typescript-operations` plugins. This gives us types, but not the custom hooks.

We have these generated types:

- `GetUserQuery`: The type of the query's result data.
- `GetUserQueryVariables`: The type of the query's variables.

And we have our query defined in a `gql` tag:

```graphql
import { gql } from '@apollo/client';

const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
    }
  }
`;
```

To use Apollo's `useQuery` hook in a type-safe way, we pass these types as generic arguments.

```jsx
import React from 'react';
import { useQuery } from '@apollo/client';
import { GetUserQuery, GetUserQueryVariables } from '../gql/generated';
// Assume GET_USER is the gql query string from above

function UserProfile({ userId }: { userId: string }) {
  // The generic signature is: `useQuery<TData, TVariables>`
  const { data, loading, error } = useQuery<GetUserQuery, GetUserQueryVariables>(
    GET_USER,
    {
      variables: { id: userId },
    }
  );

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error!</p>;

  // `data` is typed as `GetUserQuery | undefined`.
  return <div>{data?.user?.name}</div>;
}
```

This pattern works perfectly. The generics ensure that:

1.  The `data` property is correctly typed.
2.  The `variables` option in the hook's configuration must match the `GetUserQueryVariables` type. If you tried to pass `variables: { userId: userId }`, you would get a compile-time error because the schema expects `id`.

## Deep Dive

### The "Typed Hooks" Pattern (Preferred)

As we saw in the previous section, the best approach is to use the `typescript-react-apollo` plugin (or a similar one for other clients like `urql`). This generates a custom hook for every query.

**Why is this better?**

- **Less Boilerplate**: You don't need to import three things (the hook, the data type, the variables type) and wire them together with generics. You just import one ready-to-use hook.
- **Less Room for Error**: It's impossible to accidentally pass the wrong types to the wrong query, because the types are baked into the generated hook.

### Typing Mutations

The same principle applies to `useMutation`.

```jsx
import { useMutation } from '@apollo/client';
import {
  CreateUserMutation,
  CreateUserMutationVariables,
} from '../gql/generated';

const CREATE_USER = gql`
  mutation CreateUser($name: String!) {
    createUser(name: $name) {
      id
      name
    }
  }
`;

function CreateUserForm() {
  // Generic signature: `useMutation<TData, TVariables>`
  const [createUser, { data, loading, error }] = useMutation<
    CreateUserMutation,
    CreateUserMutationVariables
  >(CREATE_USER);

  const handleSubmit = () => {
    // The `variables` object here is type-checked against `CreateUserMutationVariables`.
    createUser({ variables: { name: 'New User' } });
  };

  // ...
}
```

Again, a code generator plugin would create a `useCreateUserMutation` hook that wraps all of this for you, which is the recommended approach.

### Production Perspective

- **Client-Side Schema**: Apollo Client can be configured with your schema's type information (introspection result). This, combined with tools like the Apollo Client DevTools for VS Code, can provide autocompletion and validation for your queries directly in your editor, even before you run the code generator.
- **Type Policies**: For advanced caching with Apollo Client, you'll define `TypePolicies`. These policies are also typed, allowing you to safely define how different fields in your cache are read and written.

## tRPC: End-to-End Type Safety

## Learning Objective

Understand the tRPC philosophy of creating type-safe APIs without code generation by directly inferring frontend types from the backend's TypeScript router definition.

## Why This Matters

tRPC offers a groundbreaking developer experience by providing "full-stack autocompletion." It makes calling your backend API feel like calling a local function, with full type safety and autocompletion, all without the build step of code generation. For full-stack TypeScript applications, it's a game-changer.

## Discovery Phase

The magic of tRPC starts on the backend. You define your API "router" using TypeScript.

```typescript
// FileName: server/trpc/router.ts (BACKEND CODE)
import { initTRPC } from "@trpc/server";
import { z } from "zod"; // Zod is used for input validation

const t = initTRPC.create();

export const appRouter = t.router({
  // Define a "procedure" (an endpoint)
  post: t.router({
    getById: t.procedure
      .input(z.string()) // Define input validation
      .query(({ input }) => {
        // `input` is a strongly-typed string here.
        // In a real app, you'd fetch from a database.
        return { id: input, title: "Hello from tRPC!" };
      }),
    create: t.procedure
      .input(z.object({ title: z.string(), content: z.string() }))
      .mutation(({ input }) => {
        // `input` is { title: string, content: string }
        console.log("Creating post:", input.title);
        return { id: `post-${Math.random()}`, ...input };
      }),
  }),
});

// Export only the *type* of the router. This is the key.
export type AppRouter = typeof appRouter;
```

Now, on the frontend, we set up a tRPC client. The crucial step is that we import the `AppRouter` **type**, not the router's code.

```typescript
// FileName: client/trpc.ts (FRONTEND CODE)
import { createTRPCReact } from "@trpc/react-query";
// Import the TYPE of the router from the backend.
import type { AppRouter } from "../../server/trpc/router";

// Create a typed tRPC client.
export const trpc = createTRPCReact<AppRouter>();
```

With this setup, we can now use fully-typed hooks in our components.

```jsx
// FileName: client/components/PostViewer.tsx (FRONTEND CODE)
import { trpc } from '../trpc';

function PostViewer({ postId }: { postId: string }) {
  // This feels like calling a function, but it's an API call!
  // `trpc.post.getById.useQuery(...)`
  const { data, isLoading, error } = trpc.post.getById.useQuery(postId);

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  // `data` is fully inferred as `{ id: string; title: string; }`.
  // No generics, no code generation, no manual typing.
  return <h1>{data?.title}</h1>;
}

function PostCreator() {
    const mutation = trpc.post.create.useMutation();

    const handleCreate = () => {
        // The `mutate` function's argument is typed based on the backend's
        // Zod schema: `{ title: string; content: string; }`
        mutation.mutate({ title: 'New Post', content: '...' });
    }
    // ...
}
```

If you try to call a non-existent procedure (`trpc.post.getByName.useQuery()`) or provide the wrong input type (`trpc.post.getById.useQuery(123)`), you will get an immediate TypeScript error in your editor.

## Deep Dive

### How It Works: TypeScript Inference

tRPC's "magic" is simply clever use of TypeScript inference.

1.  The backend `appRouter` is a complex object with a very specific, deeply nested type.
2.  On the frontend, we import this `AppRouter` type.
3.  The `createTRPCReact<AppRouter>()` function uses this generic type to build a client object (`trpc`) that mirrors the structure of the backend router.
4.  When you type `trpc.post.getById`, you are navigating this generated client object, and TypeScript provides autocompletion and type checking every step of the way.

It's a direct, type-level connection between your client and server code, with no intermediate steps.

### Production Perspective

- **Best for Full-Stack TypeScript**: tRPC shines in monorepos where the frontend and backend code live together, allowing for direct type imports. It's a perfect fit for frameworks like Next.js, Remix, or any project with a Node.js backend and a React frontend.
- **Not a Public API**: The main trade-off of tRPC is that it's not a language-agnostic API like REST or GraphQL. You can't easily have a Python or mobile app client consume a tRPC API. It's designed for tightly-coupled, type-safe communication within a TypeScript ecosystem.

## Zod for Runtime Validation

## Learning Objective

Use the Zod library to parse and validate untyped API responses at runtime, ensuring that the data conforms to your expected TypeScript types before it enters your application.

## Why This Matters

A type assertion (`as User`) is a promise to the compiler that you hope is true. Zod is how you _prove_ it's true. By validating data as it comes into your application, you create a secure boundary. This is the most robust way to handle data from any external source, completely eliminating errors caused by unexpected API response shapes.

## Discovery Phase

Let's go back to our initial `fetch` example. We were using a type assertion, which is risky.

```javascript
// The risky way
const userData = (await response.json()) as User;
```

Now, let's introduce Zod to make it safe.

**Step 1: Define a Zod schema.**
A schema is a definition of your data's shape and validation rules.

```javascript
// FileName: schemas/userSchema.ts
import { z } from "zod";

export const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(), // Zod has built-in validators!
  address: z.object({
    street: z.string(),
    city: z.string(),
  }),
});
```

**Step 2: Infer the TypeScript type from the schema.**
This is a key pattern: Zod becomes the single source of truth.

```javascript
// Still in schemas/userSchema.ts
import { z } from 'zod';
// ... UserSchema definition ...

// We generate our TypeScript type directly from the Zod schema.
export type User = z.infer<typeof UserSchema>;
/*
  This generates a type identical to:
  type User = {
    id: number;
    name: string;
    email: string;
    address: { street: string; city: string; };
  }
*/
```

**Step 3: Use the schema to parse the data.**
Now we update our fetch logic to use the schema.

```jsx
import { UserSchema, type User } from '../schemas/userSchema';

async function fetchAndValidateUser(): Promise<User> {
  const response = await fetch('https://jsonplaceholder.typicode.com/users/1');
  const jsonData = await response.json();

  // `parse` will throw a detailed error if the data doesn't match the schema.
  // If it succeeds, it returns the data, now guaranteed to be of type `User`.
  const user = UserSchema.parse(jsonData);

  return user;
}
```

This is bulletproof.

- If the API returns the correct shape, `UserSchema.parse` returns the fully-typed `user` data.
- If the API returns something unexpected (e.g., `name` is missing, `id` is a string), `UserSchema.parse` will throw an error that you can catch, preventing the malformed data from ever reaching your components.

## Deep Dive

### `parse` vs. `safeParse`

Zod offers two ways to validate:

- `schema.parse(data)`: Throws an error on failure. Good for `try/catch` blocks.
- `schema.safeParse(data)`: Returns an object `{ success: true, data: ... }` or `{ success: false, error: ... }`. This doesn't throw and is great for handling validation failures as part of your normal control flow.

```javascript
import { UserSchema } from './userSchema';

function processData(data: unknown) {
  const result = UserSchema.safeParse(data);

  if (result.success) {
    // `result.data` is fully typed as `User`.
    console.log('Valid user:', result.data.name);
  } else {
    // `result.error` contains detailed information about the validation failure.
    console.error('Validation failed:', result.error.flatten().fieldErrors);
  }
}
```

### Production Perspective

- **The Application Boundary**: Use Zod (or a similar library like Valibot or Yup) at the boundaries of your application. This includes API responses, form inputs, URL query parameters, and data from `localStorage`. Treat all incoming data as `unknown` until it has been successfully parsed by a schema.
- **Single Source of Truth**: The `z.infer` pattern is extremely powerful. It ensures your static TypeScript types can never drift out of sync with your runtime validation rules. Your Zod schema becomes the canonical definition of your data structures.
- **Error Handling**: The detailed errors provided by Zod are invaluable for debugging and for providing user feedback. You can use the `error.flatten()` method to get a simple object of field errors, which is perfect for displaying validation messages next to form inputs.

## Type-Safe Form Handling with Actions

## Learning Objective

Integrate Zod validation with React 19 Server Actions to create an end-to-end type-safe form submission and validation workflow.

## Why This Matters

This pattern is the culmination of modern React and TypeScript practices. It combines server-side logic (Server Actions), robust validation (Zod), and type-safe state management (`useActionState`) to create forms that are secure, reliable, and provide an excellent developer experience. It's the state-of-the-art for form handling in full-stack frameworks like Next.js.

## Discovery Phase

Let's build a simple user registration form with server-side validation.

```javascript
// FileName: schemas/registrationSchema.ts
import { z } from "zod";

export const registrationSchema = z.object({
  email: z.string().email({ message: "Invalid email address" }),
  password: z
    .string()
    .min(8, { message: "Password must be at least 8 characters" }),
});
```

Now, we'll create the Server Action that uses this schema.

```javascript
// FileName: actions/authActions.ts
'use server';
import { registrationSchema } from '../schemas/registrationSchema';

// Define the shape of the state our action will return.
export type FormState = {
  message: string;
  errors?: Record<keyof z.infer<typeof registrationSchema>, string[]>;
};

export async function registerUserAction(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  // 1. Convert FormData to a plain object.
  const rawData = Object.fromEntries(formData);

  // 2. Validate using `safeParse`.
  const validationResult = registrationSchema.safeParse(rawData);

  // 3. If validation fails, return the errors.
  if (!validationResult.success) {
    return {
      message: 'Validation failed. Please check your input.',
      errors: validationResult.error.flatten().fieldErrors,
    };
  }

  // 4. If validation succeeds, `data` is fully typed!
  const { email, password } = validationResult.data;
  console.log(`Registering user: ${email}`);
  // ... logic to save user to database ...

  return { message: 'User registered successfully!' };
}
```

Finally, let's wire this up in our client component.

```jsx
// FileName: components/RegistrationForm.tsx
'use client';
import { useActionState } from 'react';
import { registerUserAction, type FormState } from '../actions/authActions';

const initialState: FormState = { message: '' };

export function RegistrationForm() {
  const [state, formAction, isPending] = useActionState(registerUserAction, initialState);

  return (
    <form action={formAction}>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" />
        {/* Display field-specific errors from the server */}
        {state.errors?.email && <p style={{ color: 'red' }}>{state.errors.email}</p>}
      </div>
      <div>
        <label htmlFor="password">Password</label>
        <input id="password" name="password" type="password" />
        {state.errors?.password && <p style={{ color: 'red' }}>{state.errors.password}</p>}
      </div>
      <button type="submit" disabled={isPending}>
        {isPending ? 'Registering...' : 'Register'}
      </button>
      {state.message && <p>{state.message}</p>}
    </form>
  );
}
```

This creates a complete, end-to-end type-safe loop:

1.  The Zod schema is the single source of truth for validation rules and types.
2.  The Server Action uses the schema to safely validate untrusted `FormData`.
3.  The action returns a typed state object, including structured errors if validation fails.
4.  The `useActionState` hook in the client component receives this typed state, allowing us to safely access `state.errors` and display feedback to the user.

### Production Perspective

- **Security**: The most critical aspect of this pattern is that validation happens on the server. Never trust client-side validation for security. This pattern ensures that even if a user bypasses client-side checks, your server will reject invalid data.
- **Progressive Enhancement**: Because this is built on top of a standard `<form>` element, it works even if JavaScript fails to load. The form will still submit to the Server Action, and the server will respond (though without the client-side interactivity).
- **Single Source of Truth**: The Zod schema can be used in multiple places. You can use it for the server-side action validation, and you could also use it for client-side validation with a library like `react-hook-form` to provide instant feedback to the user before they even submit.

## Module Synthesis 📋

## Module Synthesis

In this chapter, we have built a robust bridge between our type-safe React application and the unpredictable world of external data. We've established a comprehensive set of patterns for fetching, validating, and managing API data with confidence and precision.

We began by learning how to **type REST API responses**, using type assertions as a first step and creating **type-safe fetch wrappers** to make our data-fetching code DRY. We then graduated to the industry-standard **React Query**, seeing how its generic hooks provide a declarative and powerful way to manage server state, caching, and mutations.

We explored the GraphQL ecosystem, understanding how **GraphQL Code Generation** provides the ultimate form of type safety by creating types and hooks directly from the API schema, and how to integrate these with **Apollo Client**. We also peeked into the future with **tRPC**, which achieves end-to-end type safety through pure TypeScript inference, eliminating the build step entirely.

Crucially, we learned that static types are not enough. We mastered **Zod for runtime validation**, creating a secure boundary for our application and ensuring that data conforms to our types before we use it. We culminated by combining these concepts into the state-of-the-art pattern for **type-safe form handling with React 19 Actions**, creating a secure, progressively enhanced, and fully-typed workflow from the browser to the server.

## Looking Ahead

You have now completed the final part of the TypeScript Integration section. You are equipped with the knowledge to build truly professional, full-stack TypeScript applications, managing types from your database schema, through your API layer, into your state management, and all the way to your React components.

The next part of this course, **Part V: Enterprise and Production**, will build on this foundation. We will move from the "how" to the "why" and "where," exploring architectural patterns, build tooling, security, and accessibility. The next chapter, **Chapter 26: Enterprise TypeScript/React Patterns**, will look at how to organize and scale these type-safe patterns in the context of large, collaborative projects, including monorepo setups and advanced architectural strategies.
