# Chapter 6: Forms and Validation

## Handling form state

## The Pain of Raw React Forms

Forms are the heart of user interaction on the web, but in React, they can be deceptively complex. A simple form requires managing state for each input, handling user changes, validating data, managing submission state, and displaying errors. Doing this manually with React's built-in hooks like `useState` is a rite of passage, but it quickly reveals the underlying challenges.

To understand why dedicated form libraries are essential, we must first walk the difficult path of building a form "the hard way." This experience will illuminate the specific problems that modern libraries are designed to solve.

### Phase 1: Establish the Reference Implementation

Our anchor example for this chapter will be a `RegistrationForm`. It's a classic, realistic use case that involves multiple input types, complex validation rules, and asynchronous submission.

Let's build the first, most naive version of this form using only the `useState` hook. We'll manage the entire form's data in a single state object.

**Project Structure**:
```
src/
└── components/
    └── RegistrationForm.tsx  ← Our reference implementation
```

```tsx
// src/components/RegistrationForm.tsx

import React, { useState } from 'react';

export function RegistrationForm() {
  const [formData, setFormData] = useState({
    fullName: '',
    email: '',
    password: '',
    confirmPassword: '',
    acceptedTerms: false,
  });

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const { name, value, type, checked } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: type === 'checkbox' ? checked : value,
    }));
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    // Basic validation
    if (formData.password !== formData.confirmPassword) {
      alert("Passwords do not match!");
      return;
    }
    if (!formData.acceptedTerms) {
      alert("You must accept the terms and conditions!");
      return;
    }
    console.log('Form Submitted:', formData);
    alert('Registration successful!');
  };

  return (
    <form onSubmit={handleSubmit} style={{ display: 'flex', flexDirection: 'column', gap: '10px', maxWidth: '400px' }}>
      <h2>Register</h2>
      <input name="fullName" value={formData.fullName} onChange={handleChange} placeholder="Full Name" />
      <input name="email" type="email" value={formData.email} onChange={handleChange} placeholder="Email" />
      <input name="password" type="password" value={formData.password} onChange={handleChange} placeholder="Password" />
      <input name="confirmPassword" type="password" value={formData.confirmPassword} onChange={handleChange} placeholder="Confirm Password" />
      <label>
        <input name="acceptedTerms" type="checkbox" checked={formData.acceptedTerms} onChange={handleChange} />
        I accept the terms and conditions
      </label>
      <button type="submit">Register</button>
    </form>
  );
}
```

This component works. It captures user input and performs a simple validation on submit. But it hides a significant performance problem.

### Iteration 1: The Unseen Cost of Controlled Components

Our current implementation uses a pattern called **controlled components**. This means React state is the "single source of truth" for the input values. The `value` of each input is tied directly to our `formData` state object, and every keystroke calls `setFormData`, triggering a re-render of the entire component.

**New Scenario Introduction**: What happens when a user types their full name, "Leonardo da Vinci," into the form?

**Failure Demonstration**: Let's add a log to our component to see how many times it re-renders.

```tsx
// src/components/RegistrationForm.tsx (with logging)

import React, { useState, useRef } from 'react';

export function RegistrationForm() {
  const renderCount = useRef(0);
  renderCount.current++;

  const [formData, setFormData] = useState({
    fullName: '',
    email: '',
    password: '',
    confirmPassword: '',
    acceptedTerms: false,
  });

  // ... (handleChange and handleSubmit are the same)
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const { name, value, type, checked } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: type === 'checkbox' ? checked : value,
    }));
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (formData.password !== formData.confirmPassword) {
      alert("Passwords do not match!");
      return;
    }
    if (!formData.acceptedTerms) {
      alert("You must accept the terms and conditions!");
      return;
    }
    console.log('Form Submitted:', formData);
    alert('Registration successful!');
  };

  return (
    <form onSubmit={handleSubmit} style={{ display: 'flex', flexDirection: 'column', gap: '10px', maxWidth: '400px' }}>
      <h2>Register</h2>
      <div>Renders: {renderCount.current}</div>
      <input name="fullName" value={formData.fullName} onChange={handleChange} placeholder="Full Name" />
      <input name="email" type="email" value={formData.email} onChange={handleChange} placeholder="Email" />
      <input name="password" type="password" value={formData.password} onChange={handleChange} placeholder="Password" />
      <input name="confirmPassword" type="password" value={formData.confirmPassword} onChange={handleChange} placeholder="Confirm Password" />
      <label>
        <input name="acceptedTerms" type="checkbox" checked={formData.acceptedTerms} onChange={handleChange} />
        I accept the terms and conditions
      </label>
      <button type="submit">Register</button>
    </form>
  );
}
```

Now, run the application and type "Leonardo da Vinci" (18 characters) into the "Full Name" field.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The form appears to work correctly. The input field updates as you type. However, the "Renders" count rapidly increases with each keystroke.

**Browser Console Output**:
No errors are logged. The form functions as expected from a user's perspective.

**React DevTools Evidence**:
- **Profiler**: Recording a session while typing "Leonardo da Vinci" shows 19 renders (1 initial + 18 for each character). Each render is fast, but in a complex application with many components, this can lead to significant performance degradation.
- **Components Tab**: The `formData` state object updates on every single keystroke.

**Let's parse this evidence**:

1.  **What the user experiences**: A functional form.
    -   Expected: The input field should update.
    -   Actual: The input field updates.

2.  **What the console reveals**: Nothing. This is a performance issue, not a runtime error.

3.  **What DevTools shows**: The entire `RegistrationForm` component re-renders every time a single character is typed into any input field.

4.  **Root cause identified**: The controlled component pattern, by its nature, ties input changes to React state updates, which forces re-renders.

5.  **Why the current approach can't solve this**: To be a controlled component, we *must* call `setState` in `onChange`. There is no way around the re-renders with this pattern. For a small form, it's acceptable. For a large form inside a complex app, it's a performance bottleneck.

6.  **What we need**: A way to manage form inputs without triggering a re-render of the entire component on every keystroke. We need to let the DOM handle the input state and only sync with React when necessary (e.g., on submission). This is the concept of **uncontrolled components**.

This is just the first crack in the foundation. As we add more complex validation and error handling, the manual `useState` approach will become exponentially more difficult to manage.

## React Hook Form: stop reinventing the wheel

## Uncontrolled Components for Peak Performance

Our `useState`-based form suffers from excessive re-renders. The solution is to adopt an **uncontrolled components** strategy, where the DOM is the source of truth for input values during typing, and React only reads the values when it needs them.

Implementing this manually is possible but involves extensive use of `useRef` for each input, which is verbose and error-prone. This is precisely the problem that `react-hook-form` was built to solve. It provides a powerful set of hooks to manage forms using uncontrolled inputs by default, giving us maximum performance with a minimal API.

### Iteration 2: Refactoring to React Hook Form

Let's refactor our `RegistrationForm` to use `react-hook-form`.

**Step 1: Installation**

```bash
npm install react-hook-form
```

**Step 2: Refactoring the Component**

We'll replace our `useState` and manual event handlers with the `useForm` hook.

**Before** (Manual `useState`):

```tsx
// src/components/RegistrationForm.tsx (Version 1 - relevant parts)

import React, { useState, useRef } from 'react';

export function RegistrationForm() {
  const renderCount = useRef(0);
  renderCount.current++;

  const [formData, setFormData] = useState({
    fullName: '',
    email: '',
    // ... other fields
  });

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    // ... manual state update logic
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    // ... manual submission logic
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>Renders: {renderCount.current}</div>
      <input name="fullName" value={formData.fullName} onChange={handleChange} />
      {/* ... other inputs */}
    </form>
  );
}
```

**After** (Using `react-hook-form`):

```tsx
// src/components/RegistrationForm.tsx (Version 2)

import React, { useRef } from 'react';
import { useForm, SubmitHandler } from 'react-hook-form';

// Define the shape of our form data
type FormValues = {
  fullName: string;
  email: string;
  password: string;
  confirmPassword: string;
  acceptedTerms: boolean;
};

export function RegistrationForm() {
  const renderCount = useRef(0);
  renderCount.current++;

  const { register, handleSubmit, formState: { errors } } = useForm<FormValues>();

  // The 'data' argument is typed and validated
  const onSubmit: SubmitHandler<FormValues> = (data) => {
    console.log('Form Submitted:', data);
    alert('Registration successful!');
  };

  return (
    // handleSubmit will validate inputs before calling our onSubmit
    <form onSubmit={handleSubmit(onSubmit)} style={{ display: 'flex', flexDirection: 'column', gap: '10px', maxWidth: '400px' }}>
      <h2>Register</h2>
      <div>Renders: {renderCount.current}</div>

      {/* Use the 'register' function to link inputs to the form state */}
      <input {...register('fullName', { required: 'Full name is required' })} placeholder="Full Name" />
      {errors.fullName && <p style={{ color: 'red' }}>{errors.fullName.message}</p>}

      <input {...register('email', { required: 'Email is required' })} placeholder="Email" />
      {errors.email && <p style={{ color: 'red' }}>{errors.email.message}</p>}

      <input type="password" {...register('password', { required: 'Password is required' })} placeholder="Password" />
      {errors.password && <p style={{ color: 'red' }}>{errors.password.message}</p>}

      <input type="password" {...register('confirmPassword', { required: 'Please confirm your password' })} placeholder="Confirm Password" />
      {errors.confirmPassword && <p style={{ color: 'red' }}>{errors.confirmPassword.message}</p>}

      <label>
        <input type="checkbox" {...register('acceptedTerms', { required: 'You must accept the terms' })} />
        I accept the terms and conditions
      </label>
      {errors.acceptedTerms && <p style={{ color: 'red' }}>{errors.acceptedTerms.message}</p>}

      <button type="submit">Register</button>
    </form>
  );
}
```

### Verification: Performance Restored

Now, run the application again and type "Leonardo da Vinci" into the "Full Name" field.

**Expected vs. Actual Improvement**:

-   **Expected**: The component should not re-render on every keystroke. The render count should remain low.
-   **Actual**: The "Renders" count stays at `1` (or `2` with Strict Mode). The component does not re-render as you type. The input field updates smoothly because the DOM is handling its state. React Hook Form only triggers a re-render when necessary, for example, when a validation error appears or disappears.

**What did `react-hook-form` do for us?**

1.  **Performance**: It eliminated the re-render-on-keystroke problem by using uncontrolled inputs.
2.  **State Management**: It removed the need for `useState` and the manual `handleChange` function. The `useForm` hook handles all form state internally.
3.  **Validation**: It provided a simple way to declare validation rules directly on the `register` call and gives us a convenient `errors` object to display messages.
4.  **Submission**: Its `handleSubmit` wrapper function prevents the default form submission, performs validation, and then passes the clean, typed data to our `onSubmit` function.

### Limitation Preview

Our form is now fast and the code is much cleaner. However, our validation logic is still defined as inline objects within the JSX. For more complex rules, like checking if "Confirm Password" matches "Password," this can get messy. Furthermore, this validation logic is coupled to our component. What if we need to validate the same data shape on the server? We'd have to duplicate the logic.

We need a way to define our data's "schema"—its shape and rules—in a single, reusable, and portable location.

## Zod: runtime validation that doesn't suck

## Declarative Validation with a Single Source of Truth

Our form is performant, but the validation logic is scattered and not easily reusable.

```tsx
// Inline validation is hard to read and maintain
<input {...register('password', { 
  required: 'Password is required', 
  minLength: { value: 8, message: 'Password must be at least 8 characters' } 
})} />
```

A more robust approach is to define a **schema**: a declarative blueprint for our data's structure and validation rules. **Zod** is a TypeScript-first schema declaration and validation library that has become the industry standard. It allows us to define a schema once and reuse it anywhere—in our form, on our server, or in API routes.

By combining Zod with React Hook Form, we create a powerful, type-safe, and maintainable validation system.

### Iteration 3: Integrating Zod for Schema-Based Validation

**Step 1: Installation**
We need Zod and the official resolver that connects it to React Hook Form.

```bash
npm install zod @hookform/resolvers
```

**Step 2: Define the Schema**

It's best practice to define schemas in a separate file so they can be imported wherever needed.

**Project Structure**:
```
src/
├── components/
│   └── RegistrationForm.tsx
└── lib/
    └── schemas.ts          ← Our new schema file
```

```typescript
// src/lib/schemas.ts

import { z } from 'zod';

export const registrationSchema = z.object({
  fullName: z.string().min(1, 'Full name is required'),
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
  confirmPassword: z.string().min(1, 'Please confirm your password'),
  acceptedTerms: z.boolean().refine(val => val === true, {
    message: 'You must accept the terms and conditions',
  }),
})
// Use .refine for complex validation that depends on multiple fields
.refine(data => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ['confirmPassword'], // Set the error on the confirmPassword field
});

// We can infer the TypeScript type from the schema
export type RegistrationFormData = z.infer<typeof registrationSchema>;
```

This schema is now our single source of truth. It clearly defines the shape of our data, the validation rules, and the error messages. It even handles the complex "passwords must match" logic cleanly with `.refine()`. Crucially, we can also infer a TypeScript type directly from the schema, eliminating the need to maintain a separate `type` definition.

**Step 3: Connect the Schema to the Form**

Now, we'll update `RegistrationForm.tsx` to use this schema via the `zodResolver`.

**Before** (Inline validation):

```tsx
// src/components/RegistrationForm.tsx (Version 2 - relevant parts)

import { useForm } from 'react-hook-form';

type FormValues = { /* ... manual type definition ... */ };

export function RegistrationForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormValues>();

  // ...
  <input {...register('fullName', { required: 'Full name is required' })} />
  // ...
}
```

**After** (Using Zod schema):

```tsx
// src/components/RegistrationForm.tsx (Version 3)

import React, { useRef } from 'react';
import { useForm, SubmitHandler } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { registrationSchema, RegistrationFormData } from '../lib/schemas'; // Import schema and type

export function RegistrationForm() {
  const renderCount = useRef(0);
  renderCount.current++;

  const { register, handleSubmit, formState: { errors } } = useForm<RegistrationFormData>({
    resolver: zodResolver(registrationSchema), // Connect Zod to React Hook Form
  });

  const onSubmit: SubmitHandler<RegistrationFormData> = (data) => {
    console.log('Form Submitted:', data);
    alert('Registration successful!');
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} style={{ display: 'flex', flexDirection: 'column', gap: '10px', maxWidth: '400px' }}>
      <h2>Register</h2>
      <div>Renders: {renderCount.current}</div>

      {/* The 'register' calls are now clean, validation logic is in the schema */}
      <input {...register('fullName')} placeholder="Full Name" />
      {errors.fullName && <p style={{ color: 'red' }}>{errors.fullName.message}</p>}

      <input {...register('email')} placeholder="Email" />
      {errors.email && <p style={{ color: 'red' }}>{errors.email.message}</p>}

      <input type="password" {...register('password')} placeholder="Password" />
      {errors.password && <p style={{ color: 'red' }}>{errors.password.message}</p>}

      <input type="password" {...register('confirmPassword')} placeholder="Confirm Password" />
      {errors.confirmPassword && <p style={{ color: 'red' }}>{errors.confirmPassword.message}</p>}

      <label>
        <input type="checkbox" {...register('acceptedTerms')} />
        I accept the terms and conditions
      </label>
      {errors.acceptedTerms && <p style={{ color: 'red' }}>{errors.acceptedTerms.message}</p>}

      <button type="submit">Register</button>
    </form>
  );
}
```

### Verification: Clean, Maintainable, and Type-Safe

Our component is now dramatically simpler. All validation logic lives in `src/lib/schemas.ts`.

-   **Maintainability**: To change a validation rule (e.g., increase password minimum length to 10), you only edit the schema file. The component doesn't need to be touched.
-   **Reusability**: The `registrationSchema` can be imported into a Next.js API route to validate incoming data on the server, ensuring consistent validation across your entire application.
-   **Type Safety**: The `RegistrationFormData` type is automatically generated from the schema. If you add a new field to the schema, TypeScript will immediately tell you if you've forgotten to handle it in your `onSubmit` function or elsewhere.

### Limitation Preview

We have a beautiful, performant, and maintainable form with declarative, type-safe validation. It's almost perfect. But forms don't exist in a vacuum. They need to communicate with a server, handle network latency, and report back API-level errors to the user. Our current `onSubmit` function is synchronous and has no concept of loading states or server-side failures.

## Building a production-ready form in 20 minutes

## From UI to Application: Handling Asynchronicity and Server Errors

A production-ready form must gracefully handle the entire submission lifecycle:
1.  User clicks "Submit."
2.  The form is disabled to prevent duplicate submissions.
3.  A loading indicator appears.
4.  A request is sent to the server.
5.  On success, a confirmation message is shown.
6.  On failure (e.g., network error or server validation error), an error message is shown to the user, and the form is re-enabled.

React Hook Form provides all the necessary tools to manage this state elegantly.

### Iteration 4: The Final, Production-Ready Form

Let's add asynchronous submission logic, loading states, and server error handling to our `RegistrationForm`.

We'll simulate an API endpoint that can either succeed or fail.

```typescript
// A mock API function to simulate a network request
const mockApiRegister = (data: RegistrationFormData): Promise<{ success: boolean; message: string; errors?: { field: keyof RegistrationFormData; message: string }[] }> => {
  return new Promise((resolve) => {
    setTimeout(() => {
      // Simulate a server-side validation error
      if (data.email === 'test@test.com') {
        resolve({
          success: false,
          message: 'Validation failed',
          errors: [{ field: 'email', message: 'This email is already taken.' }],
        });
      } else {
        resolve({ success: true, message: 'User registered successfully!' });
      }
    }, 1500); // Simulate 1.5 second network delay
  });
};
```

Now, let's integrate this into our component. We'll use the `isSubmitting` state from `formState` and the `setError` function from `useForm`.

```tsx
// src/components/RegistrationForm.tsx (Version 4 - Final)

import React, { useRef, useState } from 'react';
import { useForm, SubmitHandler } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { registrationSchema, RegistrationFormData } from '../lib/schemas';

// (mockApiRegister function would be here or imported)
const mockApiRegister = (data: RegistrationFormData): Promise<{ success: boolean; message: string; errors?: { field: keyof RegistrationFormData; message: string }[] }> => {
  return new Promise((resolve) => {
    setTimeout(() => {
      if (data.email === 'test@test.com') {
        resolve({
          success: false,
          message: 'Validation failed',
          errors: [{ field: 'email', message: 'This email is already taken.' }],
        });
      } else {
        resolve({ success: true, message: 'User registered successfully!' });
      }
    }, 1500);
  });
};

export function RegistrationForm() {
  const renderCount = useRef(0);
  renderCount.current++;

  const [serverMessage, setServerMessage] = useState<{ type: 'success' | 'error'; message: string } | null>(null);

  const { register, handleSubmit, setError, formState: { errors, isSubmitting } } = useForm<RegistrationFormData>({
    resolver: zodResolver(registrationSchema),
  });

  const onSubmit: SubmitHandler<RegistrationFormData> = async (data) => {
    setServerMessage(null);
    try {
      const response = await mockApiRegister(data);
      if (!response.success) {
        setServerMessage({ type: 'error', message: response.message });
        // Set server-side errors on specific fields
        response.errors?.forEach(err => {
          setError(err.field, { type: 'server', message: err.message });
        });
      } else {
        setServerMessage({ type: 'success', message: response.message });
      }
    } catch (error) {
      setServerMessage({ type: 'error', message: 'An unexpected error occurred. Please try again.' });
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} style={{ display: 'flex', flexDirection: 'column', gap: '10px', maxWidth: '400px' }}>
      <h2>Register</h2>
      <div>Renders: {renderCount.current}</div>

      <input {...register('fullName')} placeholder="Full Name" disabled={isSubmitting} />
      {errors.fullName && <p style={{ color: 'red' }}>{errors.fullName.message}</p>}

      <input {...register('email')} placeholder="Email" disabled={isSubmitting} />
      {errors.email && <p style={{ color: 'red' }}>{errors.email.message}</p>}

      <input type="password" {...register('password')} placeholder="Password" disabled={isSubmitting} />
      {errors.password && <p style={{ color: 'red' }}>{errors.password.message}</p>}

      <input type="password" {...register('confirmPassword')} placeholder="Confirm Password" disabled={isSubmitting} />
      {errors.confirmPassword && <p style={{ color: 'red' }}>{errors.confirmPassword.message}</p>}

      <label>
        <input type="checkbox" {...register('acceptedTerms')} disabled={isSubmitting} />
        I accept the terms and conditions
      </label>
      {errors.acceptedTerms && <p style={{ color: 'red' }}>{errors.acceptedTerms.message}</p>}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Registering...' : 'Register'}
      </button>

      {serverMessage && (
        <p style={{ color: serverMessage.type === 'error' ? 'red' : 'green' }}>
          {serverMessage.message}
        </p>
      )}
    </form>
  );
}
```

### Verification: A Robust User Experience

Try submitting the form now:
1.  **With valid data (e.g., `user@example.com`)**: The button changes to "Registering...", all fields are disabled for 1.5 seconds, and then a green success message appears.
2.  **With the specific email `test@test.com`**: The form submits, but after 1.5 seconds, a red error message appears under the email field saying "This email is already taken," and a general error message appears at the bottom. The form is re-enabled for the user to correct their input.

This final version provides a complete and professional user experience.

### Common Failure Modes and Their Signatures

#### Symptom: Form reloads the page on submission.

**Browser behavior**: The entire page flashes and reloads. Form data is lost.
**Console pattern**: No errors, but console logs disappear on reload.
**Network Tab**: A document request is made for the current page URL.
**Root cause**: You are not using React Hook Form's `handleSubmit` wrapper on your `<form>`'s `onSubmit` event, or you have a button with `type="submit"` outside the RHF-controlled form. The browser is performing its default HTML form submission behavior.
**Solution**: Ensure your form tag looks like `<form onSubmit={handleSubmit(yourSubmitFunction)}>`.

#### Symptom: A validation error doesn't appear for a specific field.

**Browser behavior**: The user can submit the form with invalid data, or an error message never shows up for one field.
**Console pattern**: No errors.
**DevTools clues**: In the React DevTools, inspect the `useForm` hook's state. You'll see the `errors` object does not contain an entry for the problematic field.
**Root cause**: Most commonly, the `name` you passed to the `register('fieldName')` function does not exactly match the property name in your Zod schema or `FormValues` type.
**Solution**: Double-check for typos. `register('fullName')` must correspond to a `fullName` property in your schema.

#### Symptom: Server-side error is not displayed in the form.

**Browser behavior**: The form submission fails, a generic error message might appear, but the specific field (e.g., email) is not highlighted with the server's error message.
**Console pattern**: You might see a 4xx or 5xx error in the network response, and your `console.log` inside the `onSubmit` function shows the error response from the server.
**DevTools clues**: The `errors` object in the `useForm` state does not contain the server error.
**Root cause**: You are not calling the `setError` function inside the `catch` block or failure condition of your submission handler.
**Solution**: After receiving an error from your API, parse the response and use `setError('fieldName', { type: 'server', message: 'Your error message' })` to programmatically add the error to the form state.

### Debugging Workflow: When Your Form Fails

**Step 1: Observe the user experience**
Is the form reloading? Is it getting stuck in a loading state? Is an error message incorrect?

**Step 2: Check the console**
Look for any React warnings or runtime errors. Also, `console.log` the `data` inside your `onSubmit` handler to see what React Hook Form is actually giving you.

**Step 3: Inspect with React DevTools**
This is your most powerful tool.
-   Select the component that calls `useForm`.
-   In the "Hooks" panel, find the `useForm` state.
-   Expand `formState`. You can inspect `errors`, `isSubmitting`, `isValid`, and `dirtyFields` in real-time. This tells you exactly what the library thinks is happening. If the `errors` object is empty, the problem is not your validation rules; it's likely in your submission logic.

**Step 4: Analyze network activity**
-   Open the Network tab in your browser's DevTools.
-   Submit the form.
-   Find the request to your API. Check the status code (e.g., 200, 400, 500).
-   Inspect the "Payload" or "Request" tab to see if the browser sent the correct data.
-   Inspect the "Response" tab to see exactly what your server sent back. The structure of this response must match what your `onSubmit` function expects to handle.

### The Journey: From Problem to Solution

| Iteration | Failure Mode                               | Technique Applied          | Result                                                              | Performance Impact                               |
| :-------- | :----------------------------------------- | :------------------------- | :------------------------------------------------------------------ | :----------------------------------------------- |
| 0         | Excessive re-renders on every keystroke.   | Raw `useState`             | Functional but slow and hard to maintain.                           | Re-renders on every input change.                |
| 1         | Cumbersome state and validation logic.     | `react-hook-form`          | Performant (uncontrolled), simplified state, but inline validation. | No re-renders on input change.                   |
| 2         | Scattered, non-reusable validation rules.  | `zod` + `zodResolver`      | Declarative, type-safe, reusable schema. Clean component.           | No change (still highly performant).             |
| 3         | No handling of async state or server errors. | `isSubmitting`, `setError` | Production-ready form with loading states and server validation.    | Re-renders only when submission state changes. |

### Final Implementation

The combination of React Hook Form and Zod provides a robust, performant, and developer-friendly pattern for building forms in modern React applications. By starting with the "raw" approach, we've seen exactly which problems these libraries solve and why they are indispensable tools in a professional developer's toolkit.
