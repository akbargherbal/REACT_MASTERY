# Chapter 6: Forms and Validation

## Handling form state

## The Problem: Forms Are Deceptively Complex

Forms seem simple. A few inputs, a submit button, maybe some validation. How hard could it be?

Let's find out by building a user settings form for our dashboard. Users should be able to update their profile information: name, email, bio, and notification preferences.

### Phase 1: The Naive Approach

Here's what most developers try first—managing form state manually with `useState`:

```tsx
// src/components/UserSettingsForm.tsx
import { useState, FormEvent } from 'react';

interface UserSettings {
  name: string;
  email: string;
  bio: string;
  emailNotifications: boolean;
  pushNotifications: boolean;
}

export function UserSettingsForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [bio, setBio] = useState('');
  const [emailNotifications, setEmailNotifications] = useState(false);
  const [pushNotifications, setPushNotifications] = useState(false);
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setIsSubmitting(true);

    try {
      const response = await fetch('/api/user/settings', {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          name,
          email,
          bio,
          emailNotifications,
          pushNotifications,
        }),
      });

      if (!response.ok) throw new Error('Failed to update settings');
      
      alert('Settings updated successfully!');
    } catch (error) {
      alert('Failed to update settings');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <label htmlFor="name" className="block text-sm font-medium">
          Name
        </label>
        <input
          id="name"
          type="text"
          value={name}
          onChange={(e) => setName(e.target.value)}
          className="mt-1 block w-full rounded border p-2"
        />
      </div>

      <div>
        <label htmlFor="email" className="block text-sm font-medium">
          Email
        </label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          className="mt-1 block w-full rounded border p-2"
        />
      </div>

      <div>
        <label htmlFor="bio" className="block text-sm font-medium">
          Bio
        </label>
        <textarea
          id="bio"
          value={bio}
          onChange={(e) => setBio(e.target.value)}
          rows={4}
          className="mt-1 block w-full rounded border p-2"
        />
      </div>

      <div className="space-y-2">
        <label className="flex items-center">
          <input
            type="checkbox"
            checked={emailNotifications}
            onChange={(e) => setEmailNotifications(e.target.checked)}
            className="mr-2"
          />
          Email notifications
        </label>

        <label className="flex items-center">
          <input
            type="checkbox"
            checked={pushNotifications}
            onChange={(e) => setPushNotifications(e.target.checked)}
            className="mr-2"
          />
          Push notifications
        </label>
      </div>

      <button
        type="submit"
        disabled={isSubmitting}
        className="rounded bg-blue-600 px-4 py-2 text-white disabled:opacity-50"
      >
        {isSubmitting ? 'Saving...' : 'Save Settings'}
      </button>
    </form>
  );
}
```

This works. You can type in the fields, check the boxes, submit the form. But let's add some real-world requirements and watch it fall apart.

### The First Failure: No Validation

**Scenario**: User submits the form with an empty name or invalid email.

Let's test it:

```tsx
// In your app, render the form and try submitting with:
// - Empty name field
// - Email: "notanemail"
// - Bio: (leave empty)

<UserSettingsForm />
```

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
- Form submits successfully
- Alert shows "Settings updated successfully!"
- No indication that the data is invalid

**Browser Console Output**:
```
POST /api/user/settings 400 Bad Request
{
  "error": "Validation failed",
  "details": {
    "name": "Name is required",
    "email": "Invalid email format"
  }
}
```

**Network Tab Analysis**:
- Request sent with invalid data: `{ name: "", email: "notanemail", ... }`
- Server responds with 400 error
- Client shows success message anyway (we're not checking `response.ok` properly)

**Let's parse this evidence**:

1. **What the user experiences**: 
   - Expected: Form should prevent submission with invalid data
   - Actual: Form submits, shows success, but server rejects it

2. **What the console reveals**: 
   - Server validation is working (400 error with details)
   - Client validation is completely missing
   - Error handling in our code is broken (we throw an error but still show success)

3. **Root cause identified**: We have no client-side validation, and our error handling doesn't actually work because we're checking `response.ok` but then showing success regardless.

4. **Why the current approach can't solve this**: Adding validation manually means:
   - Writing validation logic for each field
   - Managing error state for each field
   - Coordinating when to show errors (on blur? on submit? on change?)
   - Keeping validation logic in sync with server-side rules

5. **What we need**: A systematic way to validate form data before submission.

### Iteration 1: Adding Manual Validation

Let's add validation the hard way first, so you understand why libraries exist:

```tsx
// src/components/UserSettingsForm.tsx
import { useState, FormEvent } from 'react';

interface UserSettings {
  name: string;
  email: string;
  bio: string;
  emailNotifications: boolean;
  pushNotifications: boolean;
}

interface FormErrors {
  name?: string;
  email?: string;
  bio?: string;
}

export function UserSettingsForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [bio, setBio] = useState('');
  const [emailNotifications, setEmailNotifications] = useState(false);
  const [pushNotifications, setPushNotifications] = useState(false);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [errors, setErrors] = useState<FormErrors>({}); // ← Added

  // ← Added validation function
  const validateForm = (): boolean => {
    const newErrors: FormErrors = {};

    if (!name.trim()) {
      newErrors.name = 'Name is required';
    } else if (name.length < 2) {
      newErrors.name = 'Name must be at least 2 characters';
    }

    if (!email.trim()) {
      newErrors.email = 'Email is required';
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      newErrors.email = 'Invalid email format';
    }

    if (bio.length > 500) {
      newErrors.bio = 'Bio must be less than 500 characters';
    }

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    
    // ← Added validation check
    if (!validateForm()) {
      return;
    }

    setIsSubmitting(true);

    try {
      const response = await fetch('/api/user/settings', {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          name,
          email,
          bio,
          emailNotifications,
          pushNotifications,
        }),
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.error || 'Failed to update settings');
      }
      
      alert('Settings updated successfully!');
      setErrors({}); // ← Clear errors on success
    } catch (error) {
      alert(error instanceof Error ? error.message : 'Failed to update settings');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <label htmlFor="name" className="block text-sm font-medium">
          Name
        </label>
        <input
          id="name"
          type="text"
          value={name}
          onChange={(e) => setName(e.target.value)}
          className="mt-1 block w-full rounded border p-2"
        />
        {/* ← Added error display */}
        {errors.name && (
          <p className="mt-1 text-sm text-red-600">{errors.name}</p>
        )}
      </div>

      <div>
        <label htmlFor="email" className="block text-sm font-medium">
          Email
        </label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          className="mt-1 block w-full rounded border p-2"
        />
        {/* ← Added error display */}
        {errors.email && (
          <p className="mt-1 text-sm text-red-600">{errors.email}</p>
        )}
      </div>

      <div>
        <label htmlFor="bio" className="block text-sm font-medium">
          Bio
        </label>
        <textarea
          id="bio"
          value={bio}
          onChange={(e) => setBio(e.target.value)}
          rows={4}
          className="mt-1 block w-full rounded border p-2"
        />
        {/* ← Added error display */}
        {errors.bio && (
          <p className="mt-1 text-sm text-red-600">{errors.bio}</p>
        )}
      </div>

      <div className="space-y-2">
        <label className="flex items-center">
          <input
            type="checkbox"
            checked={emailNotifications}
            onChange={(e) => setEmailNotifications(e.target.checked)}
            className="mr-2"
          />
          Email notifications
        </label>

        <label className="flex items-center">
          <input
            type="checkbox"
            checked={pushNotifications}
            onChange={(e) => setPushNotifications(e.target.checked)}
            className="mr-2"
          />
          Push notifications
        </label>
      </div>

      <button
        type="submit"
        disabled={isSubmitting}
        className="rounded bg-blue-600 px-4 py-2 text-white disabled:opacity-50"
      >
        {isSubmitting ? 'Saving...' : 'Save Settings'}
      </button>
    </form>
  );
}
```

**Improvement**: Form now validates on submit and shows error messages.

**Verification**: Try submitting with invalid data:
- Empty name → Shows "Name is required"
- Invalid email → Shows "Invalid email format"
- Form doesn't submit until all fields are valid

### The Second Failure: Validation Timing Is Wrong

**Scenario**: User types an invalid email, then clicks submit. Error appears. User fixes the email but the error message stays until they submit again.

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
- Type "notanemail" in email field
- Click submit
- Error appears: "Invalid email format"
- Fix email to "user@example.com"
- Error message still shows "Invalid email format"
- Must click submit again to clear the error

**React DevTools Evidence**:
- `UserSettingsForm` component selected
- State: `errors: { email: "Invalid email format" }`
- User types in email field
- State: `email` updates to "user@example.com"
- State: `errors` still contains `{ email: "Invalid email format" }`
- Errors only clear on next submit

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Error should disappear when field becomes valid
   - Actual: Error persists until form is submitted again

2. **What DevTools shows**:
   - Email state updates on every keystroke
   - Error state only updates on submit
   - No connection between field changes and error clearing

3. **Root cause identified**: We only validate on submit. Field changes don't trigger validation.

4. **Why the current approach can't solve this**: We need to validate on field change, but that means:
   - Calling `validateForm()` on every keystroke
   - Validating ALL fields on EVERY change (expensive)
   - Or writing per-field validation logic (more code)
   - Managing when to show errors (immediately? after blur? after first submit?)

5. **What we need**: Smart validation that knows when to validate each field.

### Iteration 2: Validate on Change (The Naive Way)

Let's try validating on every change:

```tsx
// src/components/UserSettingsForm.tsx - Validation on change
export function UserSettingsForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [bio, setBio] = useState('');
  const [emailNotifications, setEmailNotifications] = useState(false);
  const [pushNotifications, setPushNotifications] = useState(false);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [errors, setErrors] = useState<FormErrors>({});
  const [touched, setTouched] = useState<Record<string, boolean>>({}); // ← Added

  const validateForm = (): boolean => {
    const newErrors: FormErrors = {};

    if (!name.trim()) {
      newErrors.name = 'Name is required';
    } else if (name.length < 2) {
      newErrors.name = 'Name must be at least 2 characters';
    }

    if (!email.trim()) {
      newErrors.email = 'Email is required';
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      newErrors.email = 'Invalid email format';
    }

    if (bio.length > 500) {
      newErrors.bio = 'Bio must be less than 500 characters';
    }

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  // ← Added: Validate whenever fields change
  const handleNameChange = (value: string) => {
    setName(value);
    if (touched.name) {
      validateForm();
    }
  };

  const handleEmailChange = (value: string) => {
    setEmail(value);
    if (touched.email) {
      validateForm();
    }
  };

  const handleBioChange = (value: string) => {
    setBio(value);
    if (touched.bio) {
      validateForm();
    }
  };

  const handleBlur = (field: string) => {
    setTouched({ ...touched, [field]: true });
    validateForm();
  };

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    
    // Mark all fields as touched
    setTouched({ name: true, email: true, bio: true });
    
    if (!validateForm()) {
      return;
    }

    setIsSubmitting(true);

    try {
      const response = await fetch('/api/user/settings', {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          name,
          email,
          bio,
          emailNotifications,
          pushNotifications,
        }),
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.error || 'Failed to update settings');
      }
      
      alert('Settings updated successfully!');
      setErrors({});
    } catch (error) {
      alert(error instanceof Error ? error.message : 'Failed to update settings');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <label htmlFor="name" className="block text-sm font-medium">
          Name
        </label>
        <input
          id="name"
          type="text"
          value={name}
          onChange={(e) => handleNameChange(e.target.value)}
          onBlur={() => handleBlur('name')} // ← Added
          className="mt-1 block w-full rounded border p-2"
        />
        {touched.name && errors.name && ( // ← Only show if touched
          <p className="mt-1 text-sm text-red-600">{errors.name}</p>
        )}
      </div>

      <div>
        <label htmlFor="email" className="block text-sm font-medium">
          Email
        </label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => handleEmailChange(e.target.value)}
          onBlur={() => handleBlur('email')} // ← Added
          className="mt-1 block w-full rounded border p-2"
        />
        {touched.email && errors.email && ( // ← Only show if touched
          <p className="mt-1 text-sm text-red-600">{errors.email}</p>
        )}
      </div>

      <div>
        <label htmlFor="bio" className="block text-sm font-medium">
          Bio
        </label>
        <textarea
          id="bio"
          value={bio}
          onChange={(e) => handleBioChange(e.target.value)}
          onBlur={() => handleBlur('bio')} // ← Added
          rows={4}
          className="mt-1 block w-full rounded border p-2"
        />
        {touched.bio && errors.bio && ( // ← Only show if touched
          <p className="mt-1 text-sm text-red-600">{errors.bio}</p>
        )}
      </div>

      <div className="space-y-2">
        <label className="flex items-center">
          <input
            type="checkbox"
            checked={emailNotifications}
            onChange={(e) => setEmailNotifications(e.target.checked)}
            className="mr-2"
          />
          Email notifications
        </label>

        <label className="flex items-center">
          <input
            type="checkbox"
            checked={pushNotifications}
            onChange={(e) => setPushNotifications(e.target.checked)}
            className="mr-2"
          />
          Push notifications
        </label>
      </div>

      <button
        type="submit"
        disabled={isSubmitting}
        className="rounded bg-blue-600 px-4 py-2 text-white disabled:opacity-50"
      >
        {isSubmitting ? 'Saving...' : 'Save Settings'}
      </button>
    </form>
  );
}
```

**Improvement**: Errors now clear as you fix them (after the field has been touched).

**Verification**: 
- Submit form with invalid email
- Error appears
- Fix email → Error disappears immediately
- Fresh fields don't show errors until you blur them

### The Third Failure: This Code Is a Maintenance Nightmare

Look at what we've created:
- 6 state variables for 5 form fields
- Separate change handlers for each field
- Manual touched state tracking
- Validation logic duplicated between submit and change handlers
- 150+ lines of code for a simple form

**What happens when we need to add**:
- Password field with confirmation
- Phone number with formatting
- Address with multiple fields
- Async validation (check if email is already taken)
- File upload for profile picture

**The math**: Each new field requires:
- 1 `useState` for value
- 1 entry in `touched` state
- 1 change handler function
- 1 blur handler call
- 1 validation rule
- 1 error display block
- TypeScript types for all of the above

For a 10-field form, that's 70+ additions. For a 20-field form, you're looking at 140+ additions.

### Current Limitation

Manual form state management doesn't scale. We need:
1. Automatic state management for all fields
2. Smart validation timing (touched fields only)
3. Type-safe field registration
4. Built-in error handling
5. Less boilerplate

This is exactly what React Hook Form solves.

## React Hook Form: stop reinventing the wheel

## The Solution: React Hook Form

React Hook Form is a library that handles all the tedious parts of form management:
- Field registration and state management
- Validation timing (touched, dirty, submit)
- Error handling and display
- TypeScript integration
- Performance optimization (minimal re-renders)

Let's rebuild our form using React Hook Form and see the difference.

### Installation

First, install the library:

```bash
npm install react-hook-form
```

### Iteration 3: React Hook Form Basic Implementation

Here's the same form, rewritten with React Hook Form:

```tsx
// src/components/UserSettingsForm.tsx
import { useForm } from 'react-hook-form';

interface UserSettings {
  name: string;
  email: string;
  bio: string;
  emailNotifications: boolean;
  pushNotifications: boolean;
}

export function UserSettingsForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<UserSettings>({
    defaultValues: {
      name: '',
      email: '',
      bio: '',
      emailNotifications: false,
      pushNotifications: false,
    },
  });

  const onSubmit = async (data: UserSettings) => {
    try {
      const response = await fetch('/api/user/settings', {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.error || 'Failed to update settings');
      }
      
      alert('Settings updated successfully!');
    } catch (error) {
      alert(error instanceof Error ? error.message : 'Failed to update settings');
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <label htmlFor="name" className="block text-sm font-medium">
          Name
        </label>
        <input
          id="name"
          type="text"
          {...register('name', {
            required: 'Name is required',
            minLength: {
              value: 2,
              message: 'Name must be at least 2 characters',
            },
          })}
          className="mt-1 block w-full rounded border p-2"
        />
        {errors.name && (
          <p className="mt-1 text-sm text-red-600">{errors.name.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="email" className="block text-sm font-medium">
          Email
        </label>
        <input
          id="email"
          type="email"
          {...register('email', {
            required: 'Email is required',
            pattern: {
              value: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
              message: 'Invalid email format',
            },
          })}
          className="mt-1 block w-full rounded border p-2"
        />
        {errors.email && (
          <p className="mt-1 text-sm text-red-600">{errors.email.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="bio" className="block text-sm font-medium">
          Bio
        </label>
        <textarea
          id="bio"
          {...register('bio', {
            maxLength: {
              value: 500,
              message: 'Bio must be less than 500 characters',
            },
          })}
          rows={4}
          className="mt-1 block w-full rounded border p-2"
        />
        {errors.bio && (
          <p className="mt-1 text-sm text-red-600">{errors.bio.message}</p>
        )}
      </div>

      <div className="space-y-2">
        <label className="flex items-center">
          <input
            type="checkbox"
            {...register('emailNotifications')}
            className="mr-2"
          />
          Email notifications
        </label>

        <label className="flex items-center">
          <input
            type="checkbox"
            {...register('pushNotifications')}
            className="mr-2"
          />
          Push notifications
        </label>
      </div>

      <button
        type="submit"
        disabled={isSubmitting}
        className="rounded bg-blue-600 px-4 py-2 text-white disabled:opacity-50"
      >
        {isSubmitting ? 'Saving...' : 'Save Settings'}
      </button>
    </form>
  );
}
```

**What just happened?**

Compare the line counts:
- **Manual approach**: 150+ lines
- **React Hook Form**: 80 lines

We eliminated:
- ❌ 6 `useState` declarations
- ❌ 3 custom change handlers
- ❌ 1 `touched` state object
- ❌ 1 `validateForm` function
- ❌ Manual error state management

We gained:
- ✅ Automatic field registration with `register()`
- ✅ Built-in validation rules
- ✅ Type-safe form data
- ✅ Smart validation timing (validates on blur, re-validates on change)
- ✅ Automatic `isSubmitting` state

### How React Hook Form Works

Let's break down the key pieces:

#### 1. The `useForm` Hook

```tsx
const {
  register,
  handleSubmit,
  formState: { errors, isSubmitting },
} = useForm<UserSettings>({
  defaultValues: {
    name: '',
    email: '',
    // ...
  },
});
```

This hook returns everything you need:
- `register`: Function to register input fields
- `handleSubmit`: Wrapper for your submit handler
- `formState`: Contains errors, validation state, etc.

#### 2. Field Registration

```tsx
<input
  {...register('name', {
    required: 'Name is required',
    minLength: { value: 2, message: 'Name must be at least 2 characters' },
  })}
/>
```

The `register` function returns props that connect the input to React Hook Form:
- `name`: Field identifier
- `onChange`: Tracks value changes
- `onBlur`: Triggers validation
- `ref`: Accesses the DOM element

The spread operator `{...register('name')}` applies all these props at once.

#### 3. Validation Rules

React Hook Form supports built-in validation rules:
- `required`: Field must have a value
- `minLength` / `maxLength`: String length constraints
- `min` / `max`: Number constraints
- `pattern`: Regex validation
- `validate`: Custom validation function

#### 4. Submit Handling

```tsx
<form onSubmit={handleSubmit(onSubmit)}>
```

`handleSubmit` wraps your submit function and:
- Prevents default form submission
- Validates all fields
- Only calls `onSubmit` if validation passes
- Passes validated data to your function

### Verification: Testing the Form

Let's verify the behavior:

**Test 1: Submit with empty fields**
- Click submit without filling anything
- Result: "Name is required" and "Email is required" errors appear
- Form does not submit

**Test 2: Fix one field**
- Type "John" in name field
- Blur the field (click elsewhere)
- Result: Name error disappears
- Email error still shows

**Test 3: Invalid email**
- Type "notanemail" in email field
- Blur the field
- Result: "Invalid email format" error appears

**Test 4: Fix email**
- Change email to "john@example.com"
- Result: Error disappears immediately (validates on change after first blur)

**Test 5: Valid submission**
- Fill all required fields correctly
- Click submit
- Result: Form submits, API call is made, success message appears

### Performance: Why React Hook Form Is Fast

React Hook Form uses **uncontrolled components** by default. This means:

**Traditional controlled components** (our manual approach):
```tsx
<input value={name} onChange={(e) => setName(e.target.value)} />
```
- Every keystroke triggers a state update
- State update causes component re-render
- Entire form re-renders on every keystroke

**React Hook Form's uncontrolled approach**:
```tsx
<input {...register('name')} />
```
- Field value stored in DOM, not React state
- No re-render on keystroke
- Only re-renders when validation state changes

**React DevTools Evidence**:

Open React DevTools Profiler and type in both forms:

**Manual form**:
- Each keystroke: Component renders
- 10 keystrokes = 10 renders

**React Hook Form**:
- Typing: No renders
- Blur (validation): 1 render
- 10 keystrokes = 1 render (on blur)

This matters for large forms with many fields.

### Limitation Preview

This form validates on blur and re-validates on change. But what if we need:
- Async validation (check if email is already taken)
- Cross-field validation (password confirmation)
- Complex validation logic (business rules)
- Runtime type validation (ensure data matches expected shape)

React Hook Form's built-in validators are limited to simple rules. For complex validation, we need a validation schema library.

That's where Zod comes in.

## Zod: runtime validation that doesn't suck

## The Problem: Validation Logic Gets Complex

Our current validation is simple: required fields, string lengths, regex patterns. But real-world forms need more:

**Scenario 1: Password Confirmation**
- Password must match confirmation
- Can't express this with `pattern` or `minLength`

**Scenario 2: Conditional Validation**
- If "Other" is selected, text field becomes required
- Validation rules depend on other field values

**Scenario 3: Type Safety**
- Form data should match TypeScript types
- Runtime data might not match (API responses, user input)
- Need to validate at runtime AND compile time

**Scenario 4: Reusable Validation**
- Same validation rules used in multiple forms
- Same rules used on server and client
- Don't want to duplicate logic

Let's see these problems in action.

### The Fourth Failure: Password Confirmation

Let's add password change to our form:

```tsx
// src/components/UserSettingsForm.tsx - Adding password fields
import { useForm } from 'react-hook-form';

interface UserSettings {
  name: string;
  email: string;
  bio: string;
  emailNotifications: boolean;
  pushNotifications: boolean;
  password?: string;
  confirmPassword?: string;
}

export function UserSettingsForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<UserSettings>({
    defaultValues: {
      name: '',
      email: '',
      bio: '',
      emailNotifications: false,
      pushNotifications: false,
      password: '',
      confirmPassword: '',
    },
  });

  const onSubmit = async (data: UserSettings) => {
    // How do we validate password confirmation here?
    if (data.password !== data.confirmPassword) {
      alert('Passwords do not match');
      return;
    }

    try {
      const response = await fetch('/api/user/settings', {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.error || 'Failed to update settings');
      }
      
      alert('Settings updated successfully!');
    } catch (error) {
      alert(error instanceof Error ? error.message : 'Failed to update settings');
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      {/* Previous fields... */}

      <div>
        <label htmlFor="password" className="block text-sm font-medium">
          New Password (optional)
        </label>
        <input
          id="password"
          type="password"
          {...register('password', {
            minLength: {
              value: 8,
              message: 'Password must be at least 8 characters',
            },
          })}
          className="mt-1 block w-full rounded border p-2"
        />
        {errors.password && (
          <p className="mt-1 text-sm text-red-600">{errors.password.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="confirmPassword" className="block text-sm font-medium">
          Confirm Password
        </label>
        <input
          id="confirmPassword"
          type="password"
          {...register('confirmPassword')}
          className="mt-1 block w-full rounded border p-2"
        />
        {/* How do we show "passwords don't match" error here? */}
      </div>

      <button
        type="submit"
        disabled={isSubmitting}
        className="rounded bg-blue-600 px-4 py-2 text-white disabled:opacity-50"
      >
        {isSubmitting ? 'Saving...' : 'Save Settings'}
      </button>
    </form>
  );
}
```

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
- Enter password: "password123"
- Enter confirmation: "password456"
- Click submit
- Alert shows: "Passwords do not match"
- But no error appears under the confirmation field
- User doesn't know which field is wrong

**React DevTools Evidence**:
- `formState.errors` is empty
- Validation happens in `onSubmit`, not in React Hook Form
- No way to show field-specific error for cross-field validation

**Let's parse this evidence**:

1. **What the user experiences**:
   - Expected: Error message under confirmation field
   - Actual: Generic alert, no field highlighting

2. **What DevTools shows**:
   - React Hook Form doesn't know about the validation failure
   - Error is handled manually in submit function
   - Can't use React Hook Form's error display

3. **Root cause identified**: React Hook Form's built-in validators can't compare two fields.

4. **Why the current approach can't solve this**: We could use `validate` function:

```tsx
{...register('confirmPassword', {
  validate: (value) => {
    // But how do we access the password field value here?
    // We'd need to use watch() or getValues()
    // This gets messy fast
  }
})}
```

5. **What we need**: A validation schema that can:
   - Define complex validation rules
   - Access multiple field values
   - Provide clear error messages
   - Work with TypeScript types

### The Solution: Zod

Zod is a TypeScript-first schema validation library. It lets you:
- Define validation rules as a schema
- Validate data at runtime
- Infer TypeScript types from schemas
- Compose and reuse validation logic

### Installation

```bash
npm install zod @hookform/resolvers
```

`@hookform/resolvers` provides integration between React Hook Form and validation libraries like Zod.

### Iteration 4: Zod Schema Validation

Let's rebuild our form with Zod:

```tsx
// src/components/UserSettingsForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Define validation schema
const userSettingsSchema = z.object({
  name: z
    .string()
    .min(1, 'Name is required')
    .min(2, 'Name must be at least 2 characters'),
  email: z
    .string()
    .min(1, 'Email is required')
    .email('Invalid email format'),
  bio: z
    .string()
    .max(500, 'Bio must be less than 500 characters')
    .optional(),
  emailNotifications: z.boolean(),
  pushNotifications: z.boolean(),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .optional()
    .or(z.literal('')),
  confirmPassword: z.string().optional().or(z.literal('')),
}).refine(
  (data) => {
    // Cross-field validation: passwords must match
    if (data.password && data.password !== data.confirmPassword) {
      return false;
    }
    return true;
  },
  {
    message: 'Passwords do not match',
    path: ['confirmPassword'], // Show error on confirmPassword field
  }
);

// Infer TypeScript type from schema
type UserSettings = z.infer<typeof userSettingsSchema>;

export function UserSettingsForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<UserSettings>({
    resolver: zodResolver(userSettingsSchema),
    defaultValues: {
      name: '',
      email: '',
      bio: '',
      emailNotifications: false,
      pushNotifications: false,
      password: '',
      confirmPassword: '',
    },
  });

  const onSubmit = async (data: UserSettings) => {
    // Data is already validated by Zod
    // TypeScript knows the exact shape
    try {
      const response = await fetch('/api/user/settings', {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.error || 'Failed to update settings');
      }
      
      alert('Settings updated successfully!');
    } catch (error) {
      alert(error instanceof Error ? error.message : 'Failed to update settings');
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <label htmlFor="name" className="block text-sm font-medium">
          Name
        </label>
        <input
          id="name"
          type="text"
          {...register('name')}
          className="mt-1 block w-full rounded border p-2"
        />
        {errors.name && (
          <p className="mt-1 text-sm text-red-600">{errors.name.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="email" className="block text-sm font-medium">
          Email
        </label>
        <input
          id="email"
          type="email"
          {...register('email')}
          className="mt-1 block w-full rounded border p-2"
        />
        {errors.email && (
          <p className="mt-1 text-sm text-red-600">{errors.email.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="bio" className="block text-sm font-medium">
          Bio
        </label>
        <textarea
          id="bio"
          {...register('bio')}
          rows={4}
          className="mt-1 block w-full rounded border p-2"
        />
        {errors.bio && (
          <p className="mt-1 text-sm text-red-600">{errors.bio.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="password" className="block text-sm font-medium">
          New Password (optional)
        </label>
        <input
          id="password"
          type="password"
          {...register('password')}
          className="mt-1 block w-full rounded border p-2"
        />
        {errors.password && (
          <p className="mt-1 text-sm text-red-600">{errors.password.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="confirmPassword" className="block text-sm font-medium">
          Confirm Password
        </label>
        <input
          id="confirmPassword"
          type="password"
          {...register('confirmPassword')}
          className="mt-1 block w-full rounded border p-2"
        />
        {errors.confirmPassword && (
          <p className="mt-1 text-sm text-red-600">
            {errors.confirmPassword.message}
          </p>
        )}
      </div>

      <div className="space-y-2">
        <label className="flex items-center">
          <input
            type="checkbox"
            {...register('emailNotifications')}
            className="mr-2"
          />
          Email notifications
        </label>

        <label className="flex items-center">
          <input
            type="checkbox"
            {...register('pushNotifications')}
            className="mr-2"
          />
          Push notifications
        </label>
      </div>

      <button
        type="submit"
        disabled={isSubmitting}
        className="rounded bg-blue-600 px-4 py-2 text-white disabled:opacity-50"
      >
        {isSubmitting ? 'Saving...' : 'Save Settings'}
      </button>
    </form>
  );
}
```

**What changed?**

1. **Validation moved to schema**:
```tsx
const userSettingsSchema = z.object({
  name: z.string().min(1, 'Name is required').min(2, 'Name must be at least 2 characters'),
  // ...
});
```

2. **Cross-field validation with `.refine()`**:
```tsx
.refine(
  (data) => {
    if (data.password && data.password !== data.confirmPassword) {
      return false;
    }
    return true;
  },
  {
    message: 'Passwords do not match',
    path: ['confirmPassword'], // Error shows on this field
  }
)
```

3. **Type inference**:
```tsx
type UserSettings = z.infer<typeof userSettingsSchema>;
```
TypeScript type is derived from the schema. Change the schema, type updates automatically.

4. **Resolver integration**:
```tsx
const { register, handleSubmit, formState } = useForm<UserSettings>({
  resolver: zodResolver(userSettingsSchema),
  // ...
});
```

5. **Simplified field registration**:
```tsx
<input {...register('name')} />
```
No validation rules in `register()`. All validation is in the schema.

### Verification: Testing Password Confirmation

**Test 1: Mismatched passwords**
- Password: "password123"
- Confirm: "password456"
- Blur confirm field
- Result: "Passwords do not match" appears under confirm field

**Test 2: Fix confirmation**
- Change confirm to "password123"
- Result: Error disappears immediately

**Test 3: Empty password**
- Leave both password fields empty
- Submit form
- Result: No password errors (both are optional)

**Test 4: Short password**
- Password: "pass"
- Blur field
- Result: "Password must be at least 8 characters"

### Zod Schema Patterns

Let's explore common Zod patterns you'll use:

#### Basic Types

```typescript
import { z } from 'zod';

// String validation
const nameSchema = z.string()
  .min(1, 'Required')
  .max(100, 'Too long')
  .trim(); // Remove whitespace

// Number validation
const ageSchema = z.number()
  .int('Must be an integer')
  .min(18, 'Must be 18 or older')
  .max(120, 'Invalid age');

// Email validation
const emailSchema = z.string().email('Invalid email');

// URL validation
const websiteSchema = z.string().url('Invalid URL');

// Boolean
const agreeSchema = z.boolean();

// Optional fields
const bioSchema = z.string().optional();

// Nullable fields
const middleNameSchema = z.string().nullable();

// Optional OR empty string (common for form inputs)
const optionalStringSchema = z.string().optional().or(z.literal(''));
```

#### Arrays and Objects

```typescript
import { z } from 'zod';

// Array of strings
const tagsSchema = z.array(z.string()).min(1, 'At least one tag required');

// Array of objects
const addressesSchema = z.array(
  z.object({
    street: z.string(),
    city: z.string(),
    zipCode: z.string().regex(/^\d{5}$/, 'Invalid ZIP code'),
  })
);

// Nested objects
const userSchema = z.object({
  name: z.string(),
  profile: z.object({
    bio: z.string(),
    avatar: z.string().url(),
  }),
});
```

#### Enums and Literals

```typescript
import { z } from 'zod';

// Enum (one of several values)
const roleSchema = z.enum(['admin', 'user', 'guest']);

// Literal (exact value)
const acceptTermsSchema = z.literal(true);

// Union (one of several types)
const idSchema = z.union([z.string(), z.number()]);

// Discriminated union (tagged union)
const notificationSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('email'),
    email: z.string().email(),
  }),
  z.object({
    type: z.literal('sms'),
    phone: z.string(),
  }),
]);
```

#### Custom Validation

```typescript
import { z } from 'zod';

// Custom validation with .refine()
const passwordSchema = z.string()
  .min(8, 'Password must be at least 8 characters')
  .refine(
    (password) => /[A-Z]/.test(password),
    'Password must contain at least one uppercase letter'
  )
  .refine(
    (password) => /[a-z]/.test(password),
    'Password must contain at least one lowercase letter'
  )
  .refine(
    (password) => /[0-9]/.test(password),
    'Password must contain at least one number'
  );

// Transform data
const trimmedStringSchema = z.string().transform((val) => val.trim());

// Preprocess data
const dateSchema = z.preprocess(
  (val) => (typeof val === 'string' ? new Date(val) : val),
  z.date()
);
```

#### Reusable Schemas

```typescript
// src/lib/validation.ts
import { z } from 'zod';

// Base schemas
export const emailSchema = z.string().email('Invalid email format');
export const passwordSchema = z.string().min(8, 'Password must be at least 8 characters');

// Composed schemas
export const loginSchema = z.object({
  email: emailSchema,
  password: passwordSchema,
});

export const registerSchema = z.object({
  email: emailSchema,
  password: passwordSchema,
  confirmPassword: z.string(),
}).refine(
  (data) => data.password === data.confirmPassword,
  {
    message: 'Passwords do not match',
    path: ['confirmPassword'],
  }
);

// Extend schemas
export const userProfileSchema = loginSchema.extend({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  bio: z.string().max(500, 'Bio must be less than 500 characters').optional(),
});
```

### When to Apply: Zod vs. Built-in Validation

**Use Zod when**:
- Cross-field validation (password confirmation, date ranges)
- Complex business rules (age restrictions, conditional requirements)
- Reusable validation across multiple forms
- Server-side validation needs to match client-side
- Runtime type validation (API responses, user uploads)

**Use built-in React Hook Form validation when**:
- Simple field-level rules (required, min/max length)
- Prototyping or small forms
- No need for type inference
- Performance is critical (Zod adds ~10KB to bundle)

**Decision Framework**:

| Requirement | Built-in | Zod |
|-------------|----------|-----|
| Required field | ✅ | ✅ |
| Min/max length | ✅ | ✅ |
| Regex pattern | ✅ | ✅ |
| Cross-field validation | ❌ | ✅ |
| Conditional validation | ⚠️ Complex | ✅ |
| Type inference | ❌ | ✅ |
| Reusable schemas | ❌ | ✅ |
| Server/client sharing | ❌ | ✅ |
| Bundle size | 0KB | ~10KB |

### Limitation Preview

We now have a robust form with validation. But we're still missing:
- Loading initial data (edit existing settings)
- Optimistic updates (show changes immediately)
- Error recovery (what if the API call fails?)
- Accessibility (keyboard navigation, screen readers)
- User experience polish (disable submit while invalid, show character counts)

Let's build a production-ready form that handles all of these.

## Building a production-ready form in 20 minutes

## The Complete Picture: Production-Ready Form

Let's take everything we've learned and build a form that's ready for real users. We'll add:

1. **Loading initial data** - Edit existing settings
2. **Optimistic updates** - Show changes immediately
3. **Error handling** - Graceful failure recovery
4. **Accessibility** - Keyboard navigation, ARIA labels
5. **UX polish** - Character counts, disabled states, success feedback

### Project Structure

```bash
src/
├── components/
│   └── UserSettingsForm.tsx      ← Our production form
├── lib/
│   ├── validation.ts              ← Reusable Zod schemas
│   └── api.ts                     ← API client functions
└── hooks/
    └── useUserSettings.ts         ← Data fetching hook
```

### Step 1: Reusable Validation Schema

```typescript
// src/lib/validation.ts
import { z } from 'zod';

export const userSettingsSchema = z.object({
  name: z
    .string()
    .min(1, 'Name is required')
    .min(2, 'Name must be at least 2 characters')
    .max(100, 'Name must be less than 100 characters'),
  email: z
    .string()
    .min(1, 'Email is required')
    .email('Invalid email format'),
  bio: z
    .string()
    .max(500, 'Bio must be less than 500 characters')
    .optional()
    .or(z.literal('')),
  emailNotifications: z.boolean(),
  pushNotifications: z.boolean(),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
    .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
    .regex(/[0-9]/, 'Password must contain at least one number')
    .optional()
    .or(z.literal('')),
  confirmPassword: z.string().optional().or(z.literal('')),
}).refine(
  (data) => {
    if (data.password && data.password !== data.confirmPassword) {
      return false;
    }
    return true;
  },
  {
    message: 'Passwords do not match',
    path: ['confirmPassword'],
  }
);

export type UserSettings = z.infer<typeof userSettingsSchema>;
```

### Step 2: API Client Functions

```typescript
// src/lib/api.ts
import { UserSettings } from './validation';

export async function fetchUserSettings(): Promise<UserSettings> {
  const response = await fetch('/api/user/settings');
  
  if (!response.ok) {
    throw new Error('Failed to fetch user settings');
  }
  
  return response.json();
}

export async function updateUserSettings(
  settings: UserSettings
): Promise<void> {
  const response = await fetch('/api/user/settings', {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(settings),
  });
  
  if (!response.ok) {
    const errorData = await response.json();
    throw new Error(errorData.error || 'Failed to update settings');
  }
}
```

### Step 3: Data Fetching Hook

```typescript
// src/hooks/useUserSettings.ts
import { useState, useEffect } from 'react';
import { UserSettings } from '@/lib/validation';
import { fetchUserSettings } from '@/lib/api';

export function useUserSettings() {
  const [settings, setSettings] = useState<UserSettings | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetchUserSettings()
      .then((data) => {
        setSettings(data);
        setError(null);
      })
      .catch((err) => {
        setError(err instanceof Error ? err.message : 'Failed to load settings');
      })
      .finally(() => {
        setIsLoading(false);
      });
  }, []);

  return { settings, isLoading, error };
}
```

### Step 4: Production-Ready Form Component

Now let's build the complete form with all the features:

```tsx
// src/components/UserSettingsForm.tsx
import { useEffect, useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { userSettingsSchema, UserSettings } from '@/lib/validation';
import { updateUserSettings } from '@/lib/api';
import { useUserSettings } from '@/hooks/useUserSettings';

export function UserSettingsForm() {
  const { settings, isLoading: isLoadingSettings, error: loadError } = useUserSettings();
  const [submitError, setSubmitError] = useState<string | null>(null);
  const [submitSuccess, setSubmitSuccess] = useState(false);

  const {
    register,
    handleSubmit,
    reset,
    watch,
    formState: { errors, isSubmitting, isDirty, isValid },
  } = useForm<UserSettings>({
    resolver: zodResolver(userSettingsSchema),
    defaultValues: {
      name: '',
      email: '',
      bio: '',
      emailNotifications: false,
      pushNotifications: false,
      password: '',
      confirmPassword: '',
    },
    mode: 'onBlur', // Validate on blur, re-validate on change
  });

  // Load initial data when settings are fetched
  useEffect(() => {
    if (settings) {
      reset(settings);
    }
  }, [settings, reset]);

  // Watch bio field for character count
  const bio = watch('bio');
  const bioLength = bio?.length || 0;

  const onSubmit = async (data: UserSettings) => {
    setSubmitError(null);
    setSubmitSuccess(false);

    try {
      await updateUserSettings(data);
      setSubmitSuccess(true);
      
      // Clear success message after 3 seconds
      setTimeout(() => setSubmitSuccess(false), 3000);
      
      // Clear password fields after successful update
      reset({
        ...data,
        password: '',
        confirmPassword: '',
      });
    } catch (error) {
      setSubmitError(
        error instanceof Error ? error.message : 'Failed to update settings'
      );
    }
  };

  // Loading state
  if (isLoadingSettings) {
    return (
      <div className="flex items-center justify-center p-8">
        <div className="text-center">
          <div className="mb-4 h-8 w-8 animate-spin rounded-full border-4 border-blue-600 border-t-transparent"></div>
          <p className="text-gray-600">Loading your settings...</p>
        </div>
      </div>
    );
  }

  // Error state
  if (loadError) {
    return (
      <div className="rounded-lg border border-red-200 bg-red-50 p-4">
        <h3 className="font-semibold text-red-800">Failed to load settings</h3>
        <p className="mt-1 text-sm text-red-600">{loadError}</p>
        <button
          onClick={() => window.location.reload()}
          className="mt-3 rounded bg-red-600 px-4 py-2 text-sm text-white hover:bg-red-700"
        >
          Retry
        </button>
      </div>
    );
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
      {/* Success message */}
      {submitSuccess && (
        <div
          className="rounded-lg border border-green-200 bg-green-50 p-4"
          role="alert"
          aria-live="polite"
        >
          <p className="font-semibold text-green-800">
            ✓ Settings updated successfully
          </p>
        </div>
      )}

      {/* Error message */}
      {submitError && (
        <div
          className="rounded-lg border border-red-200 bg-red-50 p-4"
          role="alert"
          aria-live="assertive"
        >
          <p className="font-semibold text-red-800">Failed to update settings</p>
          <p className="mt-1 text-sm text-red-600">{submitError}</p>
        </div>
      )}

      {/* Name field */}
      <div>
        <label
          htmlFor="name"
          className="block text-sm font-medium text-gray-700"
        >
          Name <span className="text-red-500">*</span>
        </label>
        <input
          id="name"
          type="text"
          {...register('name')}
          aria-invalid={errors.name ? 'true' : 'false'}
          aria-describedby={errors.name ? 'name-error' : undefined}
          className={`mt-1 block w-full rounded-lg border p-2.5 ${
            errors.name
              ? 'border-red-300 focus:border-red-500 focus:ring-red-500'
              : 'border-gray-300 focus:border-blue-500 focus:ring-blue-500'
          }`}
        />
        {errors.name && (
          <p id="name-error" className="mt-1 text-sm text-red-600" role="alert">
            {errors.name.message}
          </p>
        )}
      </div>

      {/* Email field */}
      <div>
        <label
          htmlFor="email"
          className="block text-sm font-medium text-gray-700"
        >
          Email <span className="text-red-500">*</span>
        </label>
        <input
          id="email"
          type="email"
          {...register('email')}
          aria-invalid={errors.email ? 'true' : 'false'}
          aria-describedby={errors.email ? 'email-error' : undefined}
          className={`mt-1 block w-full rounded-lg border p-2.5 ${
            errors.email
              ? 'border-red-300 focus:border-red-500 focus:ring-red-500'
              : 'border-gray-300 focus:border-blue-500 focus:ring-blue-500'
          }`}
        />
        {errors.email && (
          <p id="email-error" className="mt-1 text-sm text-red-600" role="alert">
            {errors.email.message}
          </p>
        )}
      </div>

      {/* Bio field with character count */}
      <div>
        <div className="flex items-center justify-between">
          <label
            htmlFor="bio"
            className="block text-sm font-medium text-gray-700"
          >
            Bio
          </label>
          <span
            className={`text-sm ${
              bioLength > 500 ? 'text-red-600' : 'text-gray-500'
            }`}
            aria-live="polite"
          >
            {bioLength}/500
          </span>
        </div>
        <textarea
          id="bio"
          {...register('bio')}
          rows={4}
          aria-invalid={errors.bio ? 'true' : 'false'}
          aria-describedby={errors.bio ? 'bio-error' : 'bio-hint'}
          className={`mt-1 block w-full rounded-lg border p-2.5 ${
            errors.bio
              ? 'border-red-300 focus:border-red-500 focus:ring-red-500'
              : 'border-gray-300 focus:border-blue-500 focus:ring-blue-500'
          }`}
        />
        {errors.bio ? (
          <p id="bio-error" className="mt-1 text-sm text-red-600" role="alert">
            {errors.bio.message}
          </p>
        ) : (
          <p id="bio-hint" className="mt-1 text-sm text-gray-500">
            Tell us a bit about yourself
          </p>
        )}
      </div>

      {/* Password fields */}
      <div className="space-y-4 rounded-lg border border-gray-200 bg-gray-50 p-4">
        <h3 className="font-medium text-gray-900">Change Password</h3>
        <p className="text-sm text-gray-600">
          Leave blank to keep your current password
        </p>

        <div>
          <label
            htmlFor="password"
            className="block text-sm font-medium text-gray-700"
          >
            New Password
          </label>
          <input
            id="password"
            type="password"
            {...register('password')}
            aria-invalid={errors.password ? 'true' : 'false'}
            aria-describedby={errors.password ? 'password-error' : 'password-hint'}
            className={`mt-1 block w-full rounded-lg border p-2.5 ${
              errors.password
                ? 'border-red-300 focus:border-red-500 focus:ring-red-500'
                : 'border-gray-300 focus:border-blue-500 focus:ring-blue-500'
            }`}
          />
          {errors.password ? (
            <p id="password-error" className="mt-1 text-sm text-red-600" role="alert">
              {errors.password.message}
            </p>
          ) : (
            <p id="password-hint" className="mt-1 text-sm text-gray-500">
              Must be at least 8 characters with uppercase, lowercase, and number
            </p>
          )}
        </div>

        <div>
          <label
            htmlFor="confirmPassword"
            className="block text-sm font-medium text-gray-700"
          >
            Confirm Password
          </label>
          <input
            id="confirmPassword"
            type="password"
            {...register('confirmPassword')}
            aria-invalid={errors.confirmPassword ? 'true' : 'false'}
            aria-describedby={errors.confirmPassword ? 'confirm-error' : undefined}
            className={`mt-1 block w-full rounded-lg border p-2.5 ${
              errors.confirmPassword
                ? 'border-red-300 focus:border-red-500 focus:ring-red-500'
                : 'border-gray-300 focus:border-blue-500 focus:ring-blue-500'
            }`}
          />
          {errors.confirmPassword && (
            <p id="confirm-error" className="mt-1 text-sm text-red-600" role="alert">
              {errors.confirmPassword.message}
            </p>
          )}
        </div>
      </div>

      {/* Notification preferences */}
      <fieldset className="space-y-3">
        <legend className="text-sm font-medium text-gray-700">
          Notification Preferences
        </legend>
        
        <label className="flex items-center">
          <input
            type="checkbox"
            {...register('emailNotifications')}
            className="h-4 w-4 rounded border-gray-300 text-blue-600 focus:ring-blue-500"
          />
          <span className="ml-2 text-sm text-gray-700">
            Email notifications
          </span>
        </label>

        <label className="flex items-center">
          <input
            type="checkbox"
            {...register('pushNotifications')}
            className="h-4 w-4 rounded border-gray-300 text-blue-600 focus:ring-blue-500"
          />
          <span className="ml-2 text-sm text-gray-700">
            Push notifications
          </span>
        </label>
      </fieldset>

      {/* Submit button */}
      <div className="flex items-center justify-between border-t pt-4">
        <button
          type="button"
          onClick={() => reset(settings || undefined)}
          disabled={!isDirty || isSubmitting}
          className="rounded-lg border border-gray-300 px-4 py-2 text-sm font-medium text-gray-700 hover:bg-gray-50 disabled:cursor-not-allowed disabled:opacity-50"
        >
          Reset
        </button>

        <button
          type="submit"
          disabled={!isDirty || !isValid || isSubmitting}
          className="rounded-lg bg-blue-600 px-6 py-2 text-sm font-medium text-white hover:bg-blue-700 disabled:cursor-not-allowed disabled:opacity-50"
        >
          {isSubmitting ? (
            <span className="flex items-center">
              <svg
                className="mr-2 h-4 w-4 animate-spin"
                viewBox="0 0 24 24"
                fill="none"
              >
                <circle
                  className="opacity-25"
                  cx="12"
                  cy="12"
                  r="10"
                  stroke="currentColor"
                  strokeWidth="4"
                />
                <path
                  className="opacity-75"
                  fill="currentColor"
                  d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
                />
              </svg>
              Saving...
            </span>
          ) : (
            'Save Changes'
          )}
        </button>
      </div>
    </form>
  );
}
```

### What Makes This Production-Ready?

Let's break down the features:

#### 1. Loading Initial Data

```tsx
const { settings, isLoading, error } = useUserSettings();

useEffect(() => {
  if (settings) {
    reset(settings); // Populate form with fetched data
  }
}, [settings, reset]);
```

- Fetches existing settings on mount
- Shows loading spinner while fetching
- Populates form when data arrives
- Handles fetch errors gracefully

#### 2. Loading and Error States

```tsx
if (isLoadingSettings) {
  return <LoadingSpinner />;
}

if (loadError) {
  return <ErrorMessage error={loadError} />;
}
```

- User sees loading indicator, not empty form
- Errors are displayed clearly with retry option
- No flash of empty form before data loads

#### 3. Form State Management

```tsx
const { isDirty, isValid, isSubmitting } = formState;

<button
  type="submit"
  disabled={!isDirty || !isValid || isSubmitting}
>
  Save Changes
</button>
```

- `isDirty`: Form has unsaved changes
- `isValid`: All validation passes
- `isSubmitting`: Submit in progress
- Submit button disabled until form is valid and changed

#### 4. Success and Error Feedback

```tsx
const [submitSuccess, setSubmitSuccess] = useState(false);
const [submitError, setSubmitError] = useState<string | null>(null);

// After successful submit
setSubmitSuccess(true);
setTimeout(() => setSubmitSuccess(false), 3000);

// After failed submit
setSubmitError(error.message);
```

- Success message appears for 3 seconds
- Error messages persist until next submit
- Both use ARIA live regions for screen readers

#### 5. Character Count

```tsx
const bio = watch('bio');
const bioLength = bio?.length || 0;

<span className={bioLength > 500 ? 'text-red-600' : 'text-gray-500'}>
  {bioLength}/500
</span>
```

- Real-time character count
- Changes color when limit exceeded
- Uses `watch()` to subscribe to field changes

#### 6. Accessibility Features

```tsx
<input
  aria-invalid={errors.name ? 'true' : 'false'}
  aria-describedby={errors.name ? 'name-error' : undefined}
/>

{errors.name && (
  <p id="name-error" role="alert">
    {errors.name.message}
  </p>
)}
```

- `aria-invalid`: Marks invalid fields for screen readers
- `aria-describedby`: Links error messages to fields
- `role="alert"`: Announces errors immediately
- `aria-live`: Announces dynamic content changes
- Proper label associations with `htmlFor`

#### 7. Visual Error States

```tsx
className={`border ${
  errors.name
    ? 'border-red-300 focus:border-red-500'
    : 'border-gray-300 focus:border-blue-500'
}`}
```

- Red border for invalid fields
- Visual feedback matches validation state
- Focus states remain accessible

#### 8. Reset Functionality

```tsx
<button
  type="button"
  onClick={() => reset(settings || undefined)}
  disabled={!isDirty || isSubmitting}
>
  Reset
</button>
```

- Resets form to last saved state
- Disabled when no changes or submitting
- Clears validation errors

#### 9. Password Field Clearing

```tsx
reset({
  ...data,
  password: '',
  confirmPassword: '',
});
```

- After successful update, password fields are cleared
- Other fields retain their values
- Security best practice

### Verification: Testing the Complete Form

**Test 1: Initial load**
- Open form
- See loading spinner
- Form populates with existing data
- All fields show current values

**Test 2: Validation**
- Clear name field, blur
- See "Name is required" error
- Field border turns red
- Submit button is disabled

**Test 3: Character count**
- Type in bio field
- See character count update in real-time
- Type 501 characters
- Count turns red
- See "Bio must be less than 500 characters" error

**Test 4: Password validation**
- Enter password: "pass"
- Blur field
- See "Password must be at least 8 characters"
- Enter password: "password"
- See "Password must contain at least one uppercase letter"
- Enter password: "Password123"
- Error clears

**Test 5: Password confirmation**
- Enter password: "Password123"
- Enter confirmation: "Password456"
- Blur confirmation
- See "Passwords do not match"
- Fix confirmation to "Password123"
- Error clears immediately

**Test 6: Submit success**
- Make valid changes
- Click "Save Changes"
- Button shows "Saving..." with spinner
- Success message appears
- Password fields clear
- Other fields retain values
- Success message disappears after 3 seconds

**Test 7: Submit error**
- Disconnect network
- Make changes and submit
- See error message
- Error persists until next submit

**Test 8: Reset**
- Make changes
- Click "Reset"
- Form reverts to last saved state
- Validation errors clear

**Test 9: Accessibility**
- Navigate form with Tab key
- All fields are reachable
- Error messages are announced by screen reader
- Submit button state is announced

### Performance Characteristics

**Bundle Size Impact**:
- React Hook Form: ~8KB gzipped
- Zod: ~10KB gzipped
- Total: ~18KB for complete form solution

**Runtime Performance**:
- No re-renders on keystroke (uncontrolled inputs)
- Validation only on blur and submit
- Character count uses `watch()` (minimal re-renders)

**React DevTools Profiler Evidence**:
- Typing in name field: 0 renders
- Blur name field: 1 render (validation)
- Typing in bio field: 1 render per keystroke (character count)
- Submit form: 2 renders (isSubmitting true → false)

### Common Failure Modes and Their Signatures

#### Symptom: Form doesn't populate with initial data

**Browser behavior**: Form shows empty fields despite data being fetched

**Console pattern**:
```
Warning: A component is changing an uncontrolled input to be controlled.
```

**DevTools clues**:
- `settings` state has data
- Form `defaultValues` are empty
- `reset()` not called after data loads

**Root cause**: Missing `useEffect` to call `reset()` when data arrives

**Solution**: Add effect to populate form:
```tsx
useEffect(() => {
  if (settings) {
    reset(settings);
  }
}, [settings, reset]);
```

#### Symptom: Submit button never enables

**Browser behavior**: Button stays disabled even with valid data

**DevTools clues**:
- `formState.isValid` is `false`
- `formState.errors` is empty
- Form mode is `'onChange'`

**Root cause**: Form mode set to `'onChange'` but fields haven't been touched

**Solution**: Change mode to `'onBlur'` or `'all'`:
```tsx
useForm({
  mode: 'onBlur', // Validate on blur, re-validate on change
});
```

#### Symptom: Character count doesn't update

**Browser behavior**: Count stays at 0 while typing

**Console pattern**: No errors

**DevTools clues**:
- Field value updates in form state
- Component doesn't re-render
- `watch()` not called

**Root cause**: Not using `watch()` to subscribe to field changes

**Solution**: Use `watch()` to track field:
```tsx
const bio = watch('bio');
const bioLength = bio?.length || 0;
```

#### Symptom: Password fields don't clear after submit

**Browser behavior**: Password remains visible after successful update

**DevTools clues**:
- Submit succeeds
- Form state still contains password
- `reset()` not called with cleared passwords

**Root cause**: Not clearing sensitive fields after submit

**Solution**: Reset with cleared passwords:
```tsx
reset({
  ...data,
  password: '',
  confirmPassword: '',
});
```

### When to Apply This Pattern

**Use this complete pattern when**:
- Building user-facing forms in production
- Form has 5+ fields
- Need validation, error handling, and loading states
- Accessibility is required
- Form edits existing data

**Simplify when**:
- Prototyping or internal tools
- Form has 1-3 simple fields
- No need for loading states (no initial data fetch)
- Performance is critical (every KB matters)

**Decision Framework**:

| Feature | Simple Form | Production Form |
|---------|-------------|-----------------|
| Fields | 1-3 | 5+ |
| Validation | Built-in HTML5 | Zod schema |
| Loading state | ❌ | ✅ |
| Error handling | Basic | Comprehensive |
| Accessibility | Basic labels | Full ARIA |
| Character counts | ❌ | ✅ |
| Reset functionality | ❌ | ✅ |
| Success feedback | Alert | Inline message |
| Bundle size | ~8KB | ~18KB |
| Development time | 30 min | 2 hours |

### The Journey: From Naive to Production

| Iteration | Approach | Lines of Code | Features | Limitations |
|-----------|----------|---------------|----------|-------------|
| 0 | Manual useState | 150+ | Basic form | No validation, prop drilling |
| 1 | Manual validation | 200+ | Field validation | Validation timing wrong |
| 2 | Touched state | 250+ | Smart validation | Maintenance nightmare |
| 3 | React Hook Form | 80 | Clean, validated | No cross-field validation |
| 4 | + Zod | 100 | Type-safe, complex rules | No loading/error states |
| 5 | Production | 200 | Complete, accessible | More complex |

### Lessons Learned

**1. Don't reinvent form state management**
- React Hook Form handles 90% of form complexity
- Manual state management doesn't scale past 3-4 fields
- The library is smaller than your custom solution

**2. Validation belongs in schemas**
- Zod schemas are reusable across client and server
- Type inference eliminates duplicate type definitions
- Complex validation is easier to read and maintain

**3. Loading and error states are not optional**
- Users need feedback at every stage
- Loading states prevent confusion
- Error states enable recovery

**4. Accessibility is built in, not bolted on**
- ARIA attributes connect errors to fields
- Live regions announce dynamic changes
- Keyboard navigation must work

**5. UX polish matters**
- Character counts guide users
- Disabled states prevent invalid submissions
- Success feedback confirms actions
- Reset functionality enables exploration

**6. Performance comes from architecture**
- Uncontrolled inputs minimize re-renders
- Validation only when needed
- Watch only the fields you need

The difference between a form that works and a form that's production-ready is attention to these details. React Hook Form and Zod handle the hard parts. Your job is to connect them thoughtfully and handle the edge cases.
