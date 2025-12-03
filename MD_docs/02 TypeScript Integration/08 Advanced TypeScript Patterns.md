# Chapter 8: Advanced TypeScript Patterns

## Discriminated unions for component variants

## Discriminated unions for component variants

In Chapter 7, we built a type-safe User Profile Dashboard. It worked well for simple cases, but as our application grew, we encountered a common problem: components that need to behave differently based on their configuration.

Let's introduce our reference implementation for this chapter: a **Notification System** that displays different types of alerts to users. This will be our anchor example, and we'll refine it through multiple iterations as we discover the limitations of naive approaches.

**Project Structure**:
```
src/
├── components/
│   ├── Notification.tsx      ← Our reference implementation
│   ├── NotificationList.tsx
│   └── Dashboard.tsx
├── types/
│   └── notifications.ts
└── app/
    └── page.tsx
```

### Phase 1: The Naive Approach - Boolean Flags Everywhere

Let's start with how most developers initially approach component variants: using boolean flags.

```tsx
// src/components/Notification.tsx - Version 1 (Naive)
interface NotificationProps {
  message: string;
  isSuccess?: boolean;
  isError?: boolean;
  isWarning?: boolean;
  isInfo?: boolean;
  showIcon?: boolean;
  isDismissible?: boolean;
  onDismiss?: () => void;
  actionLabel?: string;
  onAction?: () => void;
}

export function Notification({
  message,
  isSuccess,
  isError,
  isWarning,
  isInfo,
  showIcon,
  isDismissible,
  onDismiss,
  actionLabel,
  onAction,
}: NotificationProps) {
  const getBackgroundColor = () => {
    if (isSuccess) return 'bg-green-100';
    if (isError) return 'bg-red-100';
    if (isWarning) return 'bg-yellow-100';
    if (isInfo) return 'bg-blue-100';
    return 'bg-gray-100';
  };

  const getIcon = () => {
    if (!showIcon) return null;
    if (isSuccess) return '✓';
    if (isError) return '✗';
    if (isWarning) return '⚠';
    if (isInfo) return 'ℹ';
    return null;
  };

  return (
    <div className={`p-4 rounded ${getBackgroundColor()}`}>
      <div className="flex items-start gap-3">
        {getIcon() && <span className="text-xl">{getIcon()}</span>}
        <div className="flex-1">
          <p>{message}</p>
          {actionLabel && onAction && (
            <button
              onClick={onAction}
              className="mt-2 text-sm underline"
            >
              {actionLabel}
            </button>
          )}
        </div>
        {isDismissible && onDismiss && (
          <button onClick={onDismiss} className="text-gray-500">
            ×
          </button>
        )}
      </div>
    </div>
  );
}
```

```tsx
// src/app/page.tsx - Using the naive component
'use client';

import { useState } from 'react';
import { Notification } from '@/components/Notification';

export default function DashboardPage() {
  const [notifications, setNotifications] = useState([
    { id: 1, message: 'Profile updated successfully', isSuccess: true },
    { id: 2, message: 'Failed to load user data', isError: true },
    { id: 3, message: 'Your session expires in 5 minutes', isWarning: true },
  ]);

  return (
    <div className="p-8 space-y-4">
      <h1 className="text-2xl font-bold">Dashboard</h1>
      
      {notifications.map((notif) => (
        <Notification
          key={notif.id}
          message={notif.message}
          isSuccess={notif.isSuccess}
          isError={notif.isError}
          isWarning={notif.isWarning}
          showIcon
          isDismissible
          onDismiss={() => {
            setNotifications(notifications.filter(n => n.id !== notif.id));
          }}
        />
      ))}
    </div>
  );
}
```

This works, but let's see what happens when we try to use it in more complex scenarios.

```tsx
// src/app/page.tsx - Attempting to add an action button
export default function DashboardPage() {
  return (
    <div className="p-8 space-y-4">
      {/* This compiles but creates an invalid state */}
      <Notification
        message="Payment failed"
        isError={true}
        isSuccess={true}  // ← Both error AND success?
        showIcon
        actionLabel="Retry"
        // ← Forgot onAction handler, but TypeScript doesn't complain
      />
      
      {/* This also compiles but makes no sense */}
      <Notification
        message="Click to continue"
        // ← No type specified, but has an action
        actionLabel="Continue"
        onAction={() => console.log('clicked')}
      />
    </div>
  );
}
```

### Diagnostic Analysis: Reading the Failure

**Browser Behavior**:
The component renders, but the behavior is unpredictable. The first notification shows a success background (green) because `isSuccess` is checked first in the conditional logic, even though `isError` is also true. The second notification has an action button but no visual indication of what type of notification it is.

**Browser Console Output**:
```
(No errors - this is the problem!)
```

**TypeScript Compiler Output**:
```bash
# TypeScript compiles successfully
# No type errors detected
```

**Let's parse this evidence**:

1. **What the developer experiences**: Code compiles and runs without errors, but the component behaves inconsistently. Multiple boolean flags can be true simultaneously, creating impossible states.

2. **What TypeScript reveals**: Nothing. The type system accepts any combination of boolean flags because they're all optional and independent.

3. **Root cause identified**: Boolean flags create **2^n possible states** where n is the number of flags. With 4 type flags (`isSuccess`, `isError`, `isWarning`, `isInfo`), we have 16 possible combinations, but only 4 are valid.

4. **Why the current approach can't solve this**: Optional boolean props are independent. TypeScript has no way to express "exactly one of these must be true" or "if actionLabel is provided, onAction must also be provided."

5. **What we need**: A type system that makes **invalid states unrepresentable**. We need TypeScript to enforce that only valid combinations of props can exist.

### Iteration 1: Introducing Discriminated Unions

A **discriminated union** (also called a tagged union) is a TypeScript pattern where a single property (the "discriminant" or "tag") determines which other properties are valid.

Let's refactor our notification component to use this pattern.

```typescript
// src/types/notifications.ts - Version 2
// Define each notification variant as a separate type
type SuccessNotification = {
  variant: 'success';  // ← The discriminant
  message: string;
  dismissible?: boolean;
};

type ErrorNotification = {
  variant: 'error';
  message: string;
  dismissible?: boolean;
  action?: {
    label: string;
    onClick: () => void;
  };
};

type WarningNotification = {
  variant: 'warning';
  message: string;
  dismissible?: boolean;
};

type InfoNotification = {
  variant: 'info';
  message: string;
  dismissible?: boolean;
};

// Union type: a notification is ONE of these types
export type NotificationProps = 
  | SuccessNotification 
  | ErrorNotification 
  | WarningNotification 
  | InfoNotification;
```

Now let's update our component to use this discriminated union:

```tsx
// src/components/Notification.tsx - Version 2
import type { NotificationProps } from '@/types/notifications';

export function Notification(props: NotificationProps) {
  // TypeScript knows which properties exist based on the variant
  const getBackgroundColor = () => {
    switch (props.variant) {
      case 'success': return 'bg-green-100';
      case 'error': return 'bg-red-100';
      case 'warning': return 'bg-yellow-100';
      case 'info': return 'bg-blue-100';
    }
  };

  const getIcon = () => {
    switch (props.variant) {
      case 'success': return '✓';
      case 'error': return '✗';
      case 'warning': return '⚠';
      case 'info': return 'ℹ';
    }
  };

  return (
    <div className={`p-4 rounded ${getBackgroundColor()}`}>
      <div className="flex items-start gap-3">
        <span className="text-xl">{getIcon()}</span>
        <div className="flex-1">
          <p>{props.message}</p>
          
          {/* TypeScript knows 'action' only exists on error variant */}
          {props.variant === 'error' && props.action && (
            <button
              onClick={props.action.onClick}
              className="mt-2 text-sm underline"
            >
              {props.action.label}
            </button>
          )}
        </div>
        
        {props.dismissible && (
          <button 
            onClick={() => {/* dismiss logic */}} 
            className="text-gray-500"
          >
            ×
          </button>
        )}
      </div>
    </div>
  );
}
```

Now let's try to create those invalid states again:

```tsx
// src/app/page.tsx - Attempting invalid states with discriminated unions
export default function DashboardPage() {
  return (
    <div className="p-8 space-y-4">
      {/* ✗ TypeScript error: Cannot have both 'success' and 'error' */}
      <Notification
        variant="success"
        variant="error"  // ← Error: Duplicate identifier 'variant'
        message="This won't compile"
      />
      
      {/* ✗ TypeScript error: 'action' doesn't exist on success variant */}
      <Notification
        variant="success"
        message="Payment successful"
        action={{  // ← Error: Object literal may only specify known properties
          label: "View Receipt",
          onClick: () => {}
        }}
      />
      
      {/* ✓ This is valid - error variant can have an action */}
      <Notification
        variant="error"
        message="Payment failed"
        action={{
          label: "Retry",
          onClick: () => console.log('retrying')
        }}
      />
    </div>
  );
}
```

**TypeScript Compiler Output**:
```bash
src/app/page.tsx:6:9 - error TS1117: 
An object literal cannot have multiple properties with the same name.

6         variant="error"
          ~~~~~~~

src/app/page.tsx:14:9 - error TS2353:
Object literal may only specify known properties, and 'action' 
does not exist in type 'SuccessNotification'.

14        action={{
          ~~~~~~

Found 2 errors in src/app/page.tsx
```

**Expected vs. Actual improvement**: 
- **Before**: Invalid states compiled successfully, causing runtime bugs
- **After**: Invalid states are caught at compile time, preventing bugs before they reach the browser
- **Concrete evidence**: TypeScript now prevents us from creating notifications that are both success and error, or adding action buttons to notification types that don't support them

### The Power of Exhaustiveness Checking

One of the most powerful features of discriminated unions is **exhaustiveness checking**. TypeScript can verify that you've handled all possible variants.

```tsx
// src/components/Notification.tsx - Demonstrating exhaustiveness checking
export function Notification(props: NotificationProps) {
  const getBackgroundColor = (): string => {
    switch (props.variant) {
      case 'success': return 'bg-green-100';
      case 'error': return 'bg-red-100';
      case 'warning': return 'bg-yellow-100';
      // ← Forgot 'info' case
    }
    // TypeScript error: Function lacks ending return statement 
    // and return type does not include 'undefined'
  };

  // Better: Use a helper to enforce exhaustiveness
  const assertNever = (value: never): never => {
    throw new Error(`Unhandled variant: ${value}`);
  };

  const getBackgroundColorSafe = (): string => {
    switch (props.variant) {
      case 'success': return 'bg-green-100';
      case 'error': return 'bg-red-100';
      case 'warning': return 'bg-yellow-100';
      case 'info': return 'bg-blue-100';
      default:
        // If we add a new variant and forget to handle it,
        // TypeScript will error here
        return assertNever(props.variant);
    }
  };

  return (
    <div className={`p-4 rounded ${getBackgroundColorSafe()}`}>
      {/* ... */}
    </div>
  );
}
```

Now let's add a new notification variant and see exhaustiveness checking in action:

```typescript
// src/types/notifications.ts - Adding a new variant
type SuccessNotification = {
  variant: 'success';
  message: string;
  dismissible?: boolean;
};

type ErrorNotification = {
  variant: 'error';
  message: string;
  dismissible?: boolean;
  action?: {
    label: string;
    onClick: () => void;
  };
};

type WarningNotification = {
  variant: 'warning';
  message: string;
  dismissible?: boolean;
};

type InfoNotification = {
  variant: 'info';
  message: string;
  dismissible?: boolean;
};

// New variant added
type LoadingNotification = {
  variant: 'loading';
  message: string;
};

export type NotificationProps = 
  | SuccessNotification 
  | ErrorNotification 
  | WarningNotification 
  | InfoNotification
  | LoadingNotification;  // ← Added to union
```

**TypeScript Compiler Output**:
```bash
src/components/Notification.tsx:28:16 - error TS2345:
Argument of type 'LoadingNotification' is not assignable to parameter of type 'never'.

28        return assertNever(props.variant);
                  ~~~~~~~~~~~

Found 1 error in src/components/Notification.tsx
```

TypeScript immediately tells us we forgot to handle the new `loading` variant. This is **exhaustiveness checking** in action—the compiler ensures we handle all cases.

Let's fix it:

```tsx
// src/components/Notification.tsx - Version 3 (handling all variants)
export function Notification(props: NotificationProps) {
  const assertNever = (value: never): never => {
    throw new Error(`Unhandled variant: ${value}`);
  };

  const getBackgroundColor = (): string => {
    switch (props.variant) {
      case 'success': return 'bg-green-100';
      case 'error': return 'bg-red-100';
      case 'warning': return 'bg-yellow-100';
      case 'info': return 'bg-blue-100';
      case 'loading': return 'bg-gray-100';  // ← Added
      default:
        return assertNever(props.variant);
    }
  };

  const getIcon = () => {
    switch (props.variant) {
      case 'success': return '✓';
      case 'error': return '✗';
      case 'warning': return '⚠';
      case 'info': return 'ℹ';
      case 'loading': return '⟳';  // ← Added
      default:
        return assertNever(props.variant);
    }
  };

  return (
    <div className={`p-4 rounded ${getBackgroundColor()}`}>
      <div className="flex items-start gap-3">
        <span className="text-xl">{getIcon()}</span>
        <div className="flex-1">
          <p>{props.message}</p>
          
          {props.variant === 'error' && props.action && (
            <button
              onClick={props.action.onClick}
              className="mt-2 text-sm underline"
            >
              {props.action.label}
            </button>
          )}
        </div>
        
        {props.dismissible && (
          <button className="text-gray-500">×</button>
        )}
      </div>
    </div>
  );
}
```

### Iteration 2: Narrowing with Type Guards

TypeScript's **type narrowing** automatically understands which properties are available after checking the discriminant. This is called **control flow analysis**.

```tsx
// src/components/NotificationList.tsx - Demonstrating type narrowing
import type { NotificationProps } from '@/types/notifications';
import { Notification } from './Notification';

type NotificationWithId = NotificationProps & { id: string };

export function NotificationList({ 
  notifications 
}: { 
  notifications: NotificationWithId[] 
}) {
  // Filter to only error notifications with actions
  const actionableErrors = notifications.filter((notif) => {
    // After this check, TypeScript knows notif is ErrorNotification
    if (notif.variant === 'error') {
      // TypeScript knows 'action' exists on ErrorNotification
      return notif.action !== undefined;
    }
    return false;
  });

  // TypeScript infers the type as ErrorNotification[]
  // because we filtered by variant === 'error'
  const errorCount = actionableErrors.length;

  return (
    <div className="space-y-4">
      {errorCount > 0 && (
        <div className="p-4 bg-red-50 rounded">
          <p className="font-semibold">
            {errorCount} error{errorCount > 1 ? 's' : ''} require attention
          </p>
        </div>
      )}
      
      {notifications.map((notif) => (
        <Notification key={notif.id} {...notif} />
      ))}
    </div>
  );
}
```

### Iteration 3: Complex Discriminated Unions

Let's extend our notification system to handle more complex scenarios. We'll add a notification type that requires user confirmation before dismissing.

```typescript
// src/types/notifications.ts - Version 3 (complex variants)
type SuccessNotification = {
  variant: 'success';
  message: string;
  dismissible?: boolean;
};

type ErrorNotification = {
  variant: 'error';
  message: string;
  dismissible?: boolean;
  action?: {
    label: string;
    onClick: () => void;
  };
};

type WarningNotification = {
  variant: 'warning';
  message: string;
  dismissible?: boolean;
};

type InfoNotification = {
  variant: 'info';
  message: string;
  dismissible?: boolean;
};

type LoadingNotification = {
  variant: 'loading';
  message: string;
};

// New: Confirmation required before dismissing
type ConfirmationNotification = {
  variant: 'confirmation';
  message: string;
  confirmText: string;
  cancelText: string;
  onConfirm: () => void;
  onCancel: () => void;
  severity: 'warning' | 'danger';  // ← Nested discriminated union
};

export type NotificationProps = 
  | SuccessNotification 
  | ErrorNotification 
  | WarningNotification 
  | InfoNotification
  | LoadingNotification
  | ConfirmationNotification;
```

```tsx
// src/components/Notification.tsx - Version 4 (handling confirmation)
import type { NotificationProps } from '@/types/notifications';

export function Notification(props: NotificationProps) {
  const assertNever = (value: never): never => {
    throw new Error(`Unhandled variant: ${value}`);
  };

  const getBackgroundColor = (): string => {
    switch (props.variant) {
      case 'success': return 'bg-green-100';
      case 'error': return 'bg-red-100';
      case 'warning': return 'bg-yellow-100';
      case 'info': return 'bg-blue-100';
      case 'loading': return 'bg-gray-100';
      case 'confirmation':
        // Nested discriminated union
        return props.severity === 'danger' 
          ? 'bg-red-100' 
          : 'bg-yellow-100';
      default:
        return assertNever(props.variant);
    }
  };

  const getIcon = () => {
    switch (props.variant) {
      case 'success': return '✓';
      case 'error': return '✗';
      case 'warning': return '⚠';
      case 'info': return 'ℹ';
      case 'loading': return '⟳';
      case 'confirmation': return '?';
      default:
        return assertNever(props.variant);
    }
  };

  return (
    <div className={`p-4 rounded ${getBackgroundColor()}`}>
      <div className="flex items-start gap-3">
        <span className="text-xl">{getIcon()}</span>
        <div className="flex-1">
          <p>{props.message}</p>
          
          {/* Error action button */}
          {props.variant === 'error' && props.action && (
            <button
              onClick={props.action.onClick}
              className="mt-2 text-sm underline"
            >
              {props.action.label}
            </button>
          )}
          
          {/* Confirmation buttons */}
          {props.variant === 'confirmation' && (
            <div className="mt-3 flex gap-2">
              <button
                onClick={props.onConfirm}
                className={`px-4 py-2 rounded text-white ${
                  props.severity === 'danger' 
                    ? 'bg-red-600 hover:bg-red-700' 
                    : 'bg-yellow-600 hover:bg-yellow-700'
                }`}
              >
                {props.confirmText}
              </button>
              <button
                onClick={props.onCancel}
                className="px-4 py-2 rounded bg-gray-200 hover:bg-gray-300"
              >
                {props.cancelText}
              </button>
            </div>
          )}
        </div>
        
        {/* Only show dismiss button for dismissible variants */}
        {props.variant !== 'confirmation' && 
         props.variant !== 'loading' && 
         props.dismissible && (
          <button className="text-gray-500">×</button>
        )}
      </div>
    </div>
  );
}
```

Now let's use the confirmation notification:

```tsx
// src/app/page.tsx - Using confirmation notifications
'use client';

import { useState } from 'react';
import { Notification } from '@/components/Notification';
import type { NotificationProps } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };

export default function DashboardPage() {
  const [notifications, setNotifications] = useState<NotificationWithId[]>([
    {
      id: '1',
      variant: 'success',
      message: 'Profile updated successfully',
      dismissible: true,
    },
    {
      id: '2',
      variant: 'confirmation',
      message: 'Are you sure you want to delete your account? This action cannot be undone.',
      confirmText: 'Delete Account',
      cancelText: 'Cancel',
      severity: 'danger',
      onConfirm: () => {
        console.log('Account deleted');
        setNotifications(notifications.filter(n => n.id !== '2'));
      },
      onCancel: () => {
        console.log('Deletion cancelled');
        setNotifications(notifications.filter(n => n.id !== '2'));
      },
    },
  ]);

  return (
    <div className="p-8 space-y-4">
      <h1 className="text-2xl font-bold">Dashboard</h1>
      
      {notifications.map((notif) => (
        <Notification key={notif.id} {...notif} />
      ))}
    </div>
  );
}
```

**Expected vs. Actual improvement**:
- **Before**: Confirmation logic mixed with dismissible logic, easy to create invalid states
- **After**: Confirmation notifications are a distinct type with required callbacks, impossible to create a confirmation without both confirm and cancel handlers
- **Concrete evidence**: TypeScript enforces that confirmation notifications must have `onConfirm`, `onCancel`, `confirmText`, `cancelText`, and `severity` properties

### When to Apply This Solution

**What it optimizes for**:
- Type safety: Invalid states become unrepresentable
- Maintainability: Adding new variants forces you to handle them everywhere
- Clarity: The type system documents valid combinations

**What it sacrifices**:
- Initial setup complexity: More types to define upfront
- Verbosity: More code than simple boolean flags

**When to choose this approach**:
- Components with multiple distinct behaviors (buttons, alerts, modals)
- Data that can be in one of several mutually exclusive states
- When you need exhaustiveness checking (handling all cases)
- When invalid combinations would cause bugs

**When to avoid this approach**:
- Simple components with independent boolean flags (e.g., `disabled`, `loading`)
- When variants share 90%+ of the same properties
- Prototyping phase where requirements are still changing rapidly

**Code characteristics**:
- Setup: Medium complexity (define union types upfront)
- Maintenance: Low burden (TypeScript guides you through changes)
- Performance: Zero runtime cost (types are erased at compile time)

### Limitation preview

Our notification system now has strong type safety, but we're still manually handling the discriminant checks in every function. In the next section, we'll explore **utility types** that can help us extract and manipulate these discriminated unions more elegantly.

## Utility types that actually matter

## Utility types that actually matter

TypeScript includes dozens of built-in utility types, but most developers only need to know a handful. In this section, we'll focus on the utility types that solve real problems in React applications, using our notification system as the reference implementation.

### The Problem: Extracting Specific Variants

Our notification system works well, but what if we need to work with only error notifications? Or only notifications that can be dismissed?

```tsx
// src/components/ErrorSummary.tsx - Naive approach
import type { NotificationProps } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };

export function ErrorSummary({ 
  notifications 
}: { 
  notifications: NotificationWithId[] 
}) {
  // We want only error notifications, but TypeScript doesn't know that
  const errors = notifications.filter(n => n.variant === 'error');
  
  // TypeScript error: Property 'action' does not exist on type 'NotificationWithId'
  const actionableErrors = errors.filter(e => e.action !== undefined);
  
  return (
    <div>
      {actionableErrors.map(error => (
        <div key={error.id}>
          {error.message}
          {/* TypeScript still doesn't know 'action' exists */}
          <button onClick={error.action.onClick}>
            {error.action.label}
          </button>
        </div>
      ))}
    </div>
  );
}
```

**TypeScript Compiler Output**:
```bash
src/components/ErrorSummary.tsx:14:38 - error TS2339:
Property 'action' does not exist on type 'NotificationWithId'.

14   const actionableErrors = errors.filter(e => e.action !== undefined);
                                        ~~~~~~~~

src/components/ErrorSummary.tsx:20:28 - error TS2339:
Property 'action' does not exist on type 'NotificationWithId'.

20           <button onClick={error.action.onClick}>
                              ~~~~~~~~~~~~

Found 2 errors in src/components/ErrorSummary.tsx
```

**Root cause**: TypeScript's `filter` method doesn't narrow the type. Even though we filtered by `variant === 'error'`, TypeScript still thinks `errors` is an array of all notification types.

### Iteration 1: Extract - Pulling Out Specific Union Members

The `Extract` utility type lets us pull out specific members of a union based on a condition.

```typescript
// src/types/notifications.ts - Adding helper types
type SuccessNotification = {
  variant: 'success';
  message: string;
  dismissible?: boolean;
};

type ErrorNotification = {
  variant: 'error';
  message: string;
  dismissible?: boolean;
  action?: {
    label: string;
    onClick: () => void;
  };
};

type WarningNotification = {
  variant: 'warning';
  message: string;
  dismissible?: boolean;
};

type InfoNotification = {
  variant: 'info';
  message: string;
  dismissible?: boolean;
};

type LoadingNotification = {
  variant: 'loading';
  message: string;
};

type ConfirmationNotification = {
  variant: 'confirmation';
  message: string;
  confirmText: string;
  cancelText: string;
  onConfirm: () => void;
  onCancel: () => void;
  severity: 'warning' | 'danger';
};

export type NotificationProps = 
  | SuccessNotification 
  | ErrorNotification 
  | WarningNotification 
  | InfoNotification
  | LoadingNotification
  | ConfirmationNotification;

// Extract specific variants
export type ErrorNotificationOnly = Extract<
  NotificationProps, 
  { variant: 'error' }
>;
// Result: ErrorNotification

export type DismissibleNotifications = Extract<
  NotificationProps,
  { dismissible?: boolean }
>;
// Result: SuccessNotification | ErrorNotification | WarningNotification | InfoNotification

export type NotificationsWithActions = Extract<
  NotificationProps,
  { action?: any }
>;
// Result: ErrorNotification
```

Now let's use `Extract` to fix our error summary component:

```tsx
// src/components/ErrorSummary.tsx - Version 2 (using Extract)
import type { NotificationProps, ErrorNotificationOnly } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };
type ErrorWithId = ErrorNotificationOnly & { id: string };

export function ErrorSummary({ 
  notifications 
}: { 
  notifications: NotificationWithId[] 
}) {
  // Type assertion after filtering
  const errors = notifications.filter(
    (n): n is ErrorWithId => n.variant === 'error'
  );
  
  // Now TypeScript knows errors is ErrorWithId[]
  const actionableErrors = errors.filter(e => e.action !== undefined);
  
  return (
    <div className="space-y-2">
      <h2 className="text-lg font-semibold text-red-600">
        Errors ({errors.length})
      </h2>
      {actionableErrors.map(error => (
        <div key={error.id} className="p-3 bg-red-50 rounded">
          <p>{error.message}</p>
          {error.action && (
            <button 
              onClick={error.action.onClick}
              className="mt-2 text-sm underline"
            >
              {error.action.label}
            </button>
          )}
        </div>
      ))}
    </div>
  );
}
```

**Expected vs. Actual improvement**:
- **Before**: TypeScript couldn't understand that filtered array contained only error notifications
- **After**: Using type predicate `(n): n is ErrorWithId` tells TypeScript exactly what type the filtered array contains
- **Concrete evidence**: No more type errors when accessing `error.action`

### Iteration 2: Exclude - Removing Specific Union Members

The opposite of `Extract` is `Exclude`, which removes specific members from a union.

```typescript
// src/types/notifications.ts - Using Exclude
// Get all notifications except loading
export type InteractiveNotifications = Exclude<
  NotificationProps,
  { variant: 'loading' }
>;
// Result: All notification types except LoadingNotification

// Get all notifications except confirmation and loading
export type SimpleNotifications = Exclude<
  NotificationProps,
  { variant: 'loading' | 'confirmation' }
>;
// Result: SuccessNotification | ErrorNotification | WarningNotification | InfoNotification
```

```tsx
// src/components/NotificationList.tsx - Using Exclude
import type { NotificationProps, InteractiveNotifications } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };
type InteractiveWithId = InteractiveNotifications & { id: string };

export function NotificationList({ 
  notifications 
}: { 
  notifications: NotificationWithId[] 
}) {
  // Filter out loading notifications
  const interactive = notifications.filter(
    (n): n is InteractiveWithId => n.variant !== 'loading'
  );

  return (
    <div className="space-y-4">
      {interactive.map((notif) => (
        <div key={notif.id}>
          {/* We know these notifications can be interacted with */}
          <Notification {...notif} />
        </div>
      ))}
    </div>
  );
}
```

### Iteration 3: Pick and Omit - Selecting or Removing Properties

`Pick` and `Omit` work on object properties rather than union members.

```typescript
// src/types/notifications.ts - Using Pick and Omit
// Pick only specific properties
export type NotificationPreview = Pick<
  ErrorNotification,
  'variant' | 'message'
>;
// Result: { variant: 'error'; message: string; }

// Omit specific properties
export type NotificationWithoutActions = Omit<
  ErrorNotification,
  'action'
>;
// Result: { variant: 'error'; message: string; dismissible?: boolean; }
```

Let's use these to create a notification preview component that shows only essential information:

```tsx
// src/components/NotificationPreview.tsx
import type { NotificationProps } from '@/types/notifications';

// Create a preview type that only includes variant and message
type NotificationPreview = Pick<NotificationProps, 'variant' | 'message'>;

export function NotificationPreview({ 
  variant, 
  message 
}: NotificationPreview) {
  const getIcon = () => {
    switch (variant) {
      case 'success': return '✓';
      case 'error': return '✗';
      case 'warning': return '⚠';
      case 'info': return 'ℹ';
      case 'loading': return '⟳';
      case 'confirmation': return '?';
    }
  };

  return (
    <div className="flex items-center gap-2 text-sm">
      <span>{getIcon()}</span>
      <span className="truncate">{message}</span>
    </div>
  );
}
```

### Iteration 4: Partial and Required - Making Properties Optional or Required

`Partial` makes all properties optional. `Required` makes all properties required.

```typescript
// src/types/notifications.ts - Using Partial and Required
// Make all properties optional (useful for updates)
export type NotificationUpdate = Partial<ErrorNotification>;
// Result: All properties are optional

// Make all properties required (useful for validation)
export type CompleteErrorNotification = Required<ErrorNotification>;
// Result: { 
//   variant: 'error'; 
//   message: string; 
//   dismissible: boolean;  // ← No longer optional
//   action: {              // ← No longer optional
//     label: string; 
//     onClick: () => void; 
//   }; 
// }
```

```tsx
// src/hooks/useNotifications.ts - Using Partial for updates
import { useState } from 'react';
import type { NotificationProps } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };
type NotificationUpdate = Partial<NotificationProps> & { id: string };

export function useNotifications() {
  const [notifications, setNotifications] = useState<NotificationWithId[]>([]);

  const addNotification = (notification: NotificationProps) => {
    const id = Math.random().toString(36).substr(2, 9);
    setNotifications([...notifications, { ...notification, id }]);
  };

  // Update only specific properties of a notification
  const updateNotification = (update: NotificationUpdate) => {
    setNotifications(notifications.map(notif => 
      notif.id === update.id 
        ? { ...notif, ...update }  // Merge update into existing notification
        : notif
    ));
  };

  const removeNotification = (id: string) => {
    setNotifications(notifications.filter(n => n.id !== id));
  };

  return {
    notifications,
    addNotification,
    updateNotification,
    removeNotification,
  };
}
```

```tsx
// src/app/page.tsx - Using the hook with partial updates
'use client';

import { useNotifications } from '@/hooks/useNotifications';
import { Notification } from '@/components/Notification';

export default function DashboardPage() {
  const { notifications, addNotification, updateNotification } = useNotifications();

  const handleAddError = () => {
    addNotification({
      variant: 'error',
      message: 'Failed to save changes',
      dismissible: true,
      action: {
        label: 'Retry',
        onClick: () => console.log('Retrying...'),
      },
    });
  };

  const handleUpdateMessage = (id: string) => {
    // Only update the message, keep everything else the same
    updateNotification({
      id,
      message: 'Updated message',
    });
  };

  return (
    <div className="p-8 space-y-4">
      <button 
        onClick={handleAddError}
        className="px-4 py-2 bg-blue-600 text-white rounded"
      >
        Add Error Notification
      </button>
      
      {notifications.map((notif) => (
        <div key={notif.id}>
          <Notification {...notif} />
          <button
            onClick={() => handleUpdateMessage(notif.id)}
            className="mt-2 text-sm text-blue-600"
          >
            Update Message
          </button>
        </div>
      ))}
    </div>
  );
}
```

### Iteration 5: Record - Creating Object Types with Specific Keys

`Record<Keys, Type>` creates an object type where all keys are of type `Keys` and all values are of type `Type`.

```typescript
// src/types/notifications.ts - Using Record
// Map each variant to its configuration
export type NotificationConfig = Record<
  NotificationProps['variant'],
  {
    icon: string;
    backgroundColor: string;
    textColor: string;
  }
>;

export const notificationConfig: NotificationConfig = {
  success: {
    icon: '✓',
    backgroundColor: 'bg-green-100',
    textColor: 'text-green-800',
  },
  error: {
    icon: '✗',
    backgroundColor: 'bg-red-100',
    textColor: 'text-red-800',
  },
  warning: {
    icon: '⚠',
    backgroundColor: 'bg-yellow-100',
    textColor: 'text-yellow-800',
  },
  info: {
    icon: 'ℹ',
    backgroundColor: 'bg-blue-100',
    textColor: 'text-blue-800',
  },
  loading: {
    icon: '⟳',
    backgroundColor: 'bg-gray-100',
    textColor: 'text-gray-800',
  },
  confirmation: {
    icon: '?',
    backgroundColor: 'bg-purple-100',
    textColor: 'text-purple-800',
  },
};
```

Now let's refactor our Notification component to use this configuration:

```tsx
// src/components/Notification.tsx - Version 5 (using Record config)
import type { NotificationProps } from '@/types/notifications';
import { notificationConfig } from '@/types/notifications';

export function Notification(props: NotificationProps) {
  // Get configuration based on variant
  const config = notificationConfig[props.variant];

  return (
    <div className={`p-4 rounded ${config.backgroundColor}`}>
      <div className="flex items-start gap-3">
        <span className={`text-xl ${config.textColor}`}>
          {config.icon}
        </span>
        <div className="flex-1">
          <p className={config.textColor}>{props.message}</p>
          
          {props.variant === 'error' && props.action && (
            <button
              onClick={props.action.onClick}
              className="mt-2 text-sm underline"
            >
              {props.action.label}
            </button>
          )}
          
          {props.variant === 'confirmation' && (
            <div className="mt-3 flex gap-2">
              <button
                onClick={props.onConfirm}
                className={`px-4 py-2 rounded text-white ${
                  props.severity === 'danger' 
                    ? 'bg-red-600 hover:bg-red-700' 
                    : 'bg-yellow-600 hover:bg-yellow-700'
                }`}
              >
                {props.confirmText}
              </button>
              <button
                onClick={props.onCancel}
                className="px-4 py-2 rounded bg-gray-200 hover:bg-gray-300"
              >
                {props.cancelText}
              </button>
            </div>
          )}
        </div>
        
        {props.variant !== 'confirmation' && 
         props.variant !== 'loading' && 
         props.dismissible && (
          <button className="text-gray-500">×</button>
        )}
      </div>
    </div>
  );
}
```

**Expected vs. Actual improvement**:
- **Before**: Configuration logic scattered across multiple switch statements
- **After**: Single source of truth for variant configuration, easier to maintain and extend
- **Concrete evidence**: Adding a new variant now requires updating only the `NotificationConfig` type and the `notificationConfig` object

### Iteration 6: ReturnType and Parameters - Extracting Function Types

`ReturnType` extracts the return type of a function. `Parameters` extracts the parameter types.

```typescript
// src/types/notifications.ts - Using ReturnType and Parameters
// Extract the return type of a function
type NotificationHookReturn = ReturnType<typeof useNotifications>;
// Result: {
//   notifications: NotificationWithId[];
//   addNotification: (notification: NotificationProps) => void;
//   updateNotification: (update: NotificationUpdate) => void;
//   removeNotification: (id: string) => void;
// }

// Extract parameter types
type AddNotificationParams = Parameters<NotificationHookReturn['addNotification']>;
// Result: [notification: NotificationProps]

type UpdateNotificationParams = Parameters<NotificationHookReturn['updateNotification']>;
// Result: [update: NotificationUpdate]
```

```tsx
// src/components/NotificationManager.tsx - Using extracted types
import type { ReturnType } from 'react';
import { useNotifications } from '@/hooks/useNotifications';

// Extract the hook's return type
type NotificationHookReturn = ReturnType<typeof useNotifications>;

// Create a component that accepts the hook's return value
export function NotificationManager({ 
  notificationSystem 
}: { 
  notificationSystem: NotificationHookReturn 
}) {
  const { notifications, addNotification, removeNotification } = notificationSystem;

  return (
    <div className="space-y-4">
      <div className="flex gap-2">
        <button
          onClick={() => addNotification({
            variant: 'success',
            message: 'Operation completed',
            dismissible: true,
          })}
          className="px-4 py-2 bg-green-600 text-white rounded"
        >
          Add Success
        </button>
        <button
          onClick={() => addNotification({
            variant: 'error',
            message: 'Operation failed',
            dismissible: true,
          })}
          className="px-4 py-2 bg-red-600 text-white rounded"
        >
          Add Error
        </button>
      </div>
      
      {notifications.map((notif) => (
        <div key={notif.id} className="flex items-center gap-2">
          <Notification {...notif} />
          <button
            onClick={() => removeNotification(notif.id)}
            className="text-red-600"
          >
            Remove
          </button>
        </div>
      ))}
    </div>
  );
}
```

### When to Apply These Utility Types

**Extract**:
- **Use when**: You need to work with specific members of a discriminated union
- **Example**: Filtering notifications by variant and maintaining type safety

**Exclude**:
- **Use when**: You need to remove specific members from a union
- **Example**: Getting all notifications except loading states

**Pick**:
- **Use when**: You need only a subset of an object's properties
- **Example**: Creating preview components that show limited information

**Omit**:
- **Use when**: You need most properties except a few
- **Example**: Creating types for API responses that exclude client-only properties

**Partial**:
- **Use when**: You need to make all properties optional
- **Example**: Update operations where only changed fields are provided

**Required**:
- **Use when**: You need to ensure all properties are present
- **Example**: Validation functions that require complete data

**Record**:
- **Use when**: You need an object with specific keys and consistent value types
- **Example**: Configuration objects, lookup tables, mappings

**ReturnType**:
- **Use when**: You need to reference a function's return type without duplicating it
- **Example**: Passing hook return values between components

**Parameters**:
- **Use when**: You need to reference a function's parameter types
- **Example**: Creating wrapper functions that accept the same parameters

### Limitation preview

We've now mastered discriminated unions and utility types for our notification system. But what happens when we need to share this notification state across multiple components? In the next section, we'll explore how to type React Context and custom hooks properly.

## Typing context and custom hooks

## Typing context and custom hooks

Our notification system works well within a single component, but real applications need to share notification state across multiple components. This is where React Context comes in—but Context introduces new TypeScript challenges.

### The Problem: Context with Undefined Initial Value

Let's try to create a notification context using the naive approach:

```tsx
// src/contexts/NotificationContext.tsx - Naive approach
'use client';

import { createContext, useContext, useState, ReactNode } from 'react';
import type { NotificationProps } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };

type NotificationContextType = {
  notifications: NotificationWithId[];
  addNotification: (notification: NotificationProps) => void;
  removeNotification: (id: string) => void;
};

// ❌ Problem: We have to provide an initial value, but we don't have one yet
const NotificationContext = createContext<NotificationContextType>({
  notifications: [],
  addNotification: () => {},  // ← Dummy function
  removeNotification: () => {}, // ← Dummy function
});

export function NotificationProvider({ children }: { children: ReactNode }) {
  const [notifications, setNotifications] = useState<NotificationWithId[]>([]);

  const addNotification = (notification: NotificationProps) => {
    const id = Math.random().toString(36).substr(2, 9);
    setNotifications([...notifications, { ...notification, id }]);
  };

  const removeNotification = (id: string) => {
    setNotifications(notifications.filter(n => n.id !== id));
  };

  return (
    <NotificationContext.Provider 
      value={{ notifications, addNotification, removeNotification }}
    >
      {children}
    </NotificationContext.Provider>
  );
}

export function useNotifications() {
  return useContext(NotificationContext);
}
```

This compiles, but there's a subtle problem. Let's try to use it:

```tsx
// src/components/SomeComponent.tsx - Using the context
'use client';

import { useNotifications } from '@/contexts/NotificationContext';

export function SomeComponent() {
  const { addNotification } = useNotifications();

  // This works, but what if we use the context outside the provider?
  const handleClick = () => {
    addNotification({
      variant: 'success',
      message: 'Button clicked',
    });
  };

  return <button onClick={handleClick}>Click me</button>;
}
```

```tsx
// src/app/page.tsx - Forgetting to wrap with provider
import { SomeComponent } from '@/components/SomeComponent';

export default function Page() {
  // ❌ Forgot to wrap with NotificationProvider
  return (
    <div>
      <SomeComponent />
    </div>
  );
}
```

**Browser Behavior**:
When you click the button, nothing happens. The notification is not added.

**Browser Console Output**:
```
(No errors - the dummy function silently does nothing)
```

**Root cause**: The dummy functions in the initial context value are called instead of the real functions from the provider. TypeScript doesn't warn us because the types match.

### Diagnostic Analysis: Reading the Failure

**Let's parse this evidence**:

1. **What the developer experiences**: Code compiles and runs, but context functions don't work when used outside the provider. No error message, just silent failure.

2. **What TypeScript reveals**: Nothing. The dummy functions have the correct type signature, so TypeScript is satisfied.

3. **Root cause identified**: We provided dummy implementations to satisfy TypeScript's requirement for an initial value, but those dummy implementations can actually be called if the context is used outside the provider.

4. **Why the current approach can't solve this**: TypeScript can't distinguish between "dummy function that should never be called" and "real function from provider."

5. **What we need**: A way to make the context value `undefined` initially, and force consumers to check for `undefined` before using it. Or better yet, throw a helpful error if the context is used outside the provider.

### Iteration 1: Context with Undefined and Runtime Check

The solution is to make the context value potentially `undefined`, and add a runtime check in the hook.

```tsx
// src/contexts/NotificationContext.tsx - Version 2 (with undefined)
'use client';

import { createContext, useContext, useState, ReactNode } from 'react';
import type { NotificationProps } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };

type NotificationContextType = {
  notifications: NotificationWithId[];
  addNotification: (notification: NotificationProps) => void;
  removeNotification: (id: string) => void;
};

// ✓ Context value can be undefined
const NotificationContext = createContext<NotificationContextType | undefined>(
  undefined
);

export function NotificationProvider({ children }: { children: ReactNode }) {
  const [notifications, setNotifications] = useState<NotificationWithId[]>([]);

  const addNotification = (notification: NotificationProps) => {
    const id = Math.random().toString(36).substr(2, 9);
    setNotifications([...notifications, { ...notification, id }]);
  };

  const removeNotification = (id: string) => {
    setNotifications(notifications.filter(n => n.id !== id));
  };

  return (
    <NotificationContext.Provider 
      value={{ notifications, addNotification, removeNotification }}
    >
      {children}
    </NotificationContext.Provider>
  );
}

export function useNotifications() {
  const context = useContext(NotificationContext);
  
  // Runtime check with helpful error message
  if (context === undefined) {
    throw new Error(
      'useNotifications must be used within a NotificationProvider'
    );
  }
  
  return context;
}
```

Now let's try to use it outside the provider:

```tsx
// src/app/page.tsx - Using context outside provider
import { SomeComponent } from '@/components/SomeComponent';

export default function Page() {
  // Still forgot to wrap with NotificationProvider
  return (
    <div>
      <SomeComponent />
    </div>
  );
}
```

**Browser Behavior**:
The page crashes immediately when `SomeComponent` tries to use the context.

**Browser Console Output**:
```
Error: useNotifications must be used within a NotificationProvider
    at useNotifications (NotificationContext.tsx:38)
    at SomeComponent (SomeComponent.tsx:6)
```

**React Error Overlay**:
```
Unhandled Runtime Error
Error: useNotifications must be used within a NotificationProvider

Source
src/contexts/NotificationContext.tsx (38:10) @ useNotifications

  36 |   const context = useContext(NotificationContext);
  37 |   if (context === undefined) {
> 38 |     throw new Error(
     |          ^
  39 |       'useNotifications must be used within a NotificationProvider'
  40 |     );
  41 |   }
```

**Expected vs. Actual improvement**:
- **Before**: Silent failure, no indication of what went wrong
- **After**: Clear error message immediately points to the problem
- **Concrete evidence**: Developer knows exactly what to fix—wrap the component tree with `NotificationProvider`

Let's fix it properly:

```tsx
// src/app/layout.tsx - Wrapping with provider
import { NotificationProvider } from '@/contexts/NotificationContext';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <NotificationProvider>
          {children}
        </NotificationProvider>
      </body>
    </html>
  );
}
```

### Iteration 2: Typing Custom Hooks with Generics

Now let's create a more sophisticated custom hook that can work with different types of notifications. We'll use generics to make it reusable.

```typescript
// src/hooks/useFilteredNotifications.ts
import { useMemo } from 'react';
import type { NotificationProps } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };

// Generic hook that filters notifications by variant
export function useFilteredNotifications<T extends NotificationProps['variant']>(
  notifications: NotificationWithId[],
  variant: T
): Extract<NotificationWithId, { variant: T }>[] {
  return useMemo(() => {
    return notifications.filter(
      (n): n is Extract<NotificationWithId, { variant: T }> => 
        n.variant === variant
    );
  }, [notifications, variant]);
}
```

Let's use this generic hook:

```tsx
// src/components/ErrorList.tsx - Using generic hook
'use client';

import { useNotifications } from '@/contexts/NotificationContext';
import { useFilteredNotifications } from '@/hooks/useFilteredNotifications';

export function ErrorList() {
  const { notifications } = useNotifications();
  
  // TypeScript infers the return type as ErrorNotification[]
  const errors = useFilteredNotifications(notifications, 'error');

  return (
    <div className="space-y-2">
      <h2 className="text-lg font-semibold text-red-600">
        Errors ({errors.length})
      </h2>
      {errors.map((error) => (
        <div key={error.id} className="p-3 bg-red-50 rounded">
          <p>{error.message}</p>
          {/* TypeScript knows 'action' exists on error variant */}
          {error.action && (
            <button 
              onClick={error.action.onClick}
              className="mt-2 text-sm underline"
            >
              {error.action.label}
            </button>
          )}
        </div>
      ))}
    </div>
  );
}
```

```tsx
// src/components/SuccessList.tsx - Same hook, different type
'use client';

import { useNotifications } from '@/contexts/NotificationContext';
import { useFilteredNotifications } from '@/hooks/useFilteredNotifications';

export function SuccessList() {
  const { notifications } = useNotifications();
  
  // TypeScript infers the return type as SuccessNotification[]
  const successes = useFilteredNotifications(notifications, 'success');

  return (
    <div className="space-y-2">
      <h2 className="text-lg font-semibold text-green-600">
        Success ({successes.length})
      </h2>
      {successes.map((success) => (
        <div key={success.id} className="p-3 bg-green-50 rounded">
          <p>{success.message}</p>
          {/* TypeScript knows 'action' does NOT exist on success variant */}
        </div>
      ))}
    </div>
  );
}
```

### Iteration 3: Complex Hook with Multiple Return Types

Let's create a more complex hook that returns different types based on its configuration.

```typescript
// src/hooks/useNotificationManager.ts
import { useState, useCallback } from 'react';
import type { NotificationProps } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };

type UseNotificationManagerOptions = {
  maxNotifications?: number;
  autoRemoveDelay?: number;
};

type NotificationManager = {
  notifications: NotificationWithId[];
  addNotification: (notification: NotificationProps) => string;
  removeNotification: (id: string) => void;
  clearAll: () => void;
  getNotificationById: (id: string) => NotificationWithId | undefined;
};

export function useNotificationManager(
  options: UseNotificationManagerOptions = {}
): NotificationManager {
  const { maxNotifications = 5, autoRemoveDelay } = options;
  const [notifications, setNotifications] = useState<NotificationWithId[]>([]);

  const addNotification = useCallback((notification: NotificationProps): string => {
    const id = Math.random().toString(36).substr(2, 9);
    const newNotification = { ...notification, id };

    setNotifications((prev) => {
      const updated = [...prev, newNotification];
      // Enforce max notifications
      if (updated.length > maxNotifications) {
        return updated.slice(-maxNotifications);
      }
      return updated;
    });

    // Auto-remove after delay if specified
    if (autoRemoveDelay) {
      setTimeout(() => {
        removeNotification(id);
      }, autoRemoveDelay);
    }

    return id;
  }, [maxNotifications, autoRemoveDelay]);

  const removeNotification = useCallback((id: string) => {
    setNotifications((prev) => prev.filter((n) => n.id !== id));
  }, []);

  const clearAll = useCallback(() => {
    setNotifications([]);
  }, []);

  const getNotificationById = useCallback((id: string) => {
    return notifications.find((n) => n.id === id);
  }, [notifications]);

  return {
    notifications,
    addNotification,
    removeNotification,
    clearAll,
    getNotificationById,
  };
}
```

Now let's use this hook in a component:

```tsx
// src/components/NotificationDemo.tsx
'use client';

import { useNotificationManager } from '@/hooks/useNotificationManager';
import { Notification } from './Notification';

export function NotificationDemo() {
  const manager = useNotificationManager({
    maxNotifications: 3,
    autoRemoveDelay: 5000, // Auto-remove after 5 seconds
  });

  const handleAddSuccess = () => {
    const id = manager.addNotification({
      variant: 'success',
      message: 'Operation completed successfully',
      dismissible: true,
    });
    console.log('Added notification with id:', id);
  };

  const handleAddError = () => {
    manager.addNotification({
      variant: 'error',
      message: 'Operation failed',
      dismissible: true,
      action: {
        label: 'Retry',
        onClick: () => console.log('Retrying...'),
      },
    });
  };

  return (
    <div className="p-8 space-y-4">
      <div className="flex gap-2">
        <button
          onClick={handleAddSuccess}
          className="px-4 py-2 bg-green-600 text-white rounded"
        >
          Add Success
        </button>
        <button
          onClick={handleAddError}
          className="px-4 py-2 bg-red-600 text-white rounded"
        >
          Add Error
        </button>
        <button
          onClick={manager.clearAll}
          className="px-4 py-2 bg-gray-600 text-white rounded"
        >
          Clear All
        </button>
      </div>

      <div className="space-y-2">
        {manager.notifications.map((notif) => (
          <Notification key={notif.id} {...notif} />
        ))}
      </div>

      <p className="text-sm text-gray-600">
        Showing {manager.notifications.length} of max 3 notifications
      </p>
    </div>
  );
}
```

### Iteration 4: Typing Hooks with Overloads

Sometimes a hook needs to return different types based on its parameters. We can use function overloads for this.

```typescript
// src/hooks/useNotification.ts - Hook with overloads
import { useState, useEffect } from 'react';
import type { NotificationProps } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };

// Overload signatures
export function useNotification(id: string): NotificationWithId | null;
export function useNotification(id: string, required: true): NotificationWithId;
export function useNotification(id: string, required: false): NotificationWithId | null;

// Implementation
export function useNotification(
  id: string,
  required?: boolean
): NotificationWithId | null {
  const [notification, setNotification] = useState<NotificationWithId | null>(null);

  useEffect(() => {
    // Simulate fetching notification by id
    const fetchNotification = async () => {
      // In real app, this would be an API call
      const mockNotification: NotificationWithId = {
        id,
        variant: 'info',
        message: `Notification ${id}`,
        dismissible: true,
      };
      setNotification(mockNotification);
    };

    fetchNotification();
  }, [id]);

  if (required && notification === null) {
    throw new Error(`Notification with id ${id} is required but not found`);
  }

  return notification;
}
```

```tsx
// src/components/NotificationDetail.tsx - Using overloaded hook
'use client';

import { useNotification } from '@/hooks/useNotification';

export function NotificationDetail({ id }: { id: string }) {
  // TypeScript knows this can be null
  const notification = useNotification(id);

  if (!notification) {
    return <div>Loading...</div>;
  }

  return (
    <div className="p-4 border rounded">
      <p>{notification.message}</p>
    </div>
  );
}

export function RequiredNotificationDetail({ id }: { id: string }) {
  // TypeScript knows this is never null (will throw if not found)
  const notification = useNotification(id, true);

  // No null check needed
  return (
    <div className="p-4 border rounded">
      <p>{notification.message}</p>
    </div>
  );
}
```

### When to Apply These Patterns

**Context with undefined + runtime check**:
- **Use when**: Creating context that should only be used within a provider
- **What it optimizes for**: Clear error messages when context is misused
- **What it sacrifices**: Extra runtime check on every hook call
- **When to choose**: Always, for any context that requires a provider

**Generic hooks**:
- **Use when**: Hook logic is the same but types vary based on parameters
- **What it optimizes for**: Code reuse without sacrificing type safety
- **What it sacrifices**: Slightly more complex type signatures
- **When to choose**: When you find yourself duplicating hook logic for different types

**Hooks with overloads**:
- **Use when**: Hook behavior changes significantly based on parameters
- **What it optimizes for**: Type safety for different usage patterns
- **What it sacrifices**: More complex type definitions
- **When to choose**: When a hook has distinct modes of operation (e.g., required vs. optional)

**Code characteristics**:
- Setup: Medium complexity (requires understanding of generics and overloads)
- Maintenance: Low burden (TypeScript guides you through changes)
- Performance: Zero runtime cost (types are erased at compile time)

### Limitation preview

We've now mastered typing context and custom hooks, making our notification system fully type-safe across component boundaries. But there's one more TypeScript topic we need to address: when is it actually okay to use `any`? In the next section, we'll explore the pragmatic use of `any` and its safer alternatives.

## When to use `any` (yes, really)

## When to use `any` (yes, really)

TypeScript purists will tell you to never use `any`. They're wrong. There are legitimate cases where `any` is the pragmatic choice. The key is knowing when to use it, and more importantly, how to contain its impact.

### The Problem: Third-Party Libraries Without Types

Let's extend our notification system to integrate with a third-party analytics library that doesn't have TypeScript definitions.

```typescript
// Imagine this is from 'legacy-analytics' package
// No @types/legacy-analytics exists
declare module 'legacy-analytics' {
  export function track(event: string, properties: any): void;
  export function identify(userId: string, traits: any): void;
}
```

```tsx
// src/hooks/useNotificationAnalytics.ts - Naive approach
import { useEffect } from 'react';
import { track } from 'legacy-analytics';
import type { NotificationProps } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };

export function useNotificationAnalytics(notification: NotificationWithId) {
  useEffect(() => {
    // ❌ TypeScript error: 'notification' is not assignable to parameter of type 'any'
    // Actually, it IS assignable, but we're trying to be too strict
    track('notification_shown', {
      variant: notification.variant,
      message: notification.message,
      hasAction: 'action' in notification,
    });
  }, [notification]);
}
```

**TypeScript Compiler Output**:
```bash
src/hooks/useNotificationAnalytics.ts:12:5 - error TS2345:
Argument of type '{ variant: "success" | "error" | "warning" | "info" | "loading" | "confirmation"; message: string; hasAction: boolean; }' 
is not assignable to parameter of type 'any'.

12     track('notification_shown', {
       ~~~~~

Found 1 error in src/hooks/useNotificationAnalytics.ts
```

Wait, that error message doesn't make sense. The library accepts `any`, so why is TypeScript complaining?

### Diagnostic Analysis: Reading the Failure

**Let's parse this evidence**:

1. **What the developer experiences**: TypeScript rejects perfectly valid code because the third-party library uses `any` in its type definitions.

2. **What TypeScript reveals**: The error message is confusing—it says our object "is not assignable to parameter of type 'any'", which seems impossible since `any` accepts anything.

3. **Root cause identified**: This is actually a different issue—TypeScript is being overly strict about object literal types. The real problem is that we're trying to avoid `any` when the library explicitly uses it.

4. **Why fighting `any` here is wrong**: The library doesn't have types. We could spend hours creating type definitions for it, or we could accept that this boundary is untyped and move on.

5. **What we need**: A pragmatic approach to using `any` at library boundaries while keeping the rest of our code type-safe.

### Iteration 1: Strategic Use of `any` at Boundaries

The solution is to use `any` at the boundary with the untyped library, but keep everything else type-safe.

```tsx
// src/hooks/useNotificationAnalytics.ts - Version 2 (pragmatic)
import { useEffect } from 'react';
import { track } from 'legacy-analytics';
import type { NotificationProps } from '@/types/notifications';

type NotificationWithId = NotificationProps & { id: string };

// Helper function that explicitly accepts any for the properties
function trackNotification(event: string, properties: any): void {
  track(event, properties);
}

export function useNotificationAnalytics(notification: NotificationWithId) {
  useEffect(() => {
    // ✓ Our code is type-safe, we just pass it to the untyped boundary
    trackNotification('notification_shown', {
      variant: notification.variant,
      message: notification.message,
      hasAction: 'action' in notification,
    });
  }, [notification]);
}
```

**Expected vs. Actual improvement**:
- **Before**: Fighting TypeScript to avoid `any`, creating friction
- **After**: Accept `any` at the library boundary, keep our code type-safe
- **Concrete evidence**: Code compiles, analytics work, and we didn't waste time creating type definitions for a library we don't control

### Iteration 2: `unknown` - The Safer Alternative

When you receive data from an external source (API, localStorage, third-party library), use `unknown` instead of `any`. `unknown` forces you to validate the data before using it.

```typescript
// src/utils/storage.ts - Using unknown for external data
export function getStoredNotifications(): unknown {
  const stored = localStorage.getItem('notifications');
  if (!stored) return null;
  
  // ✓ Return unknown, not any
  return JSON.parse(stored);
}

// Type guard to validate the data
function isNotificationArray(value: unknown): value is NotificationWithId[] {
  if (!Array.isArray(value)) return false;
  
  return value.every((item) => {
    return (
      typeof item === 'object' &&
      item !== null &&
      'id' in item &&
      'variant' in item &&
      'message' in item &&
      typeof item.id === 'string' &&
      typeof item.message === 'string'
    );
  });
}

export function loadNotifications(): NotificationWithId[] {
  const stored = getStoredNotifications();
  
  // ✓ Must validate before using
  if (isNotificationArray(stored)) {
    return stored;
  }
  
  return [];
}
```

Let's see what happens if we try to use `unknown` without validation:

```typescript
// src/utils/storage.ts - Attempting to use unknown without validation
export function loadNotificationsBad(): NotificationWithId[] {
  const stored = getStoredNotifications();
  
  // ❌ TypeScript error: Type 'unknown' is not assignable to type 'NotificationWithId[]'
  return stored;
}
```

**TypeScript Compiler Output**:
```bash
src/utils/storage.ts:45:10 - error TS2322:
Type 'unknown' is not assignable to type 'NotificationWithId[]'.

45   return stored;
            ~~~~~~

Found 1 error in src/utils/storage.ts
```

This is exactly what we want—TypeScript forces us to validate the data.

### Iteration 3: `as any` - The Escape Hatch

Sometimes you need to tell TypeScript "trust me, I know what I'm doing." Use `as any` sparingly, and always leave a comment explaining why.

```tsx
// src/components/NotificationPortal.tsx - Using as any for DOM manipulation
'use client';

import { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';
import type { ReactNode } from 'react';

export function NotificationPortal({ children }: { children: ReactNode }) {
  const portalRoot = useRef<HTMLElement | null>(null);

  useEffect(() => {
    // Create portal root if it doesn't exist
    let root = document.getElementById('notification-portal');
    
    if (!root) {
      root = document.createElement('div');
      root.id = 'notification-portal';
      document.body.appendChild(root);
    }
    
    portalRoot.current = root;

    return () => {
      // Cleanup: remove portal root if it's empty
      if (root && root.childNodes.length === 0) {
        // TypeScript doesn't know that root.parentNode exists
        // because it could be null if the element was removed
        // We know it exists because we just checked root
        (root.parentNode as any)?.removeChild(root);
        // Alternative: Use type assertion with explanation
        // root.parentNode?.removeChild(root); // TypeScript error
        // (root.parentNode as HTMLElement).removeChild(root); // Better
      }
    };
  }, []);

  if (!portalRoot.current) return null;

  return createPortal(children, portalRoot.current);
}
```

Actually, let's refactor that to avoid `as any`:

```tsx
// src/components/NotificationPortal.tsx - Version 2 (better)
'use client';

import { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';
import type { ReactNode } from 'react';

export function NotificationPortal({ children }: { children: ReactNode }) {
  const portalRoot = useRef<HTMLElement | null>(null);

  useEffect(() => {
    let root = document.getElementById('notification-portal');
    
    if (!root) {
      root = document.createElement('div');
      root.id = 'notification-portal';
      document.body.appendChild(root);
    }
    
    portalRoot.current = root;

    return () => {
      // Better: Check if parentNode exists before using it
      if (root && root.childNodes.length === 0 && root.parentNode) {
        root.parentNode.removeChild(root);
      }
    };
  }, []);

  if (!portalRoot.current) return null;

  return createPortal(children, portalRoot.current);
}
```

### Iteration 4: When `any` Is Actually the Right Choice

Here are legitimate cases where `any` is the pragmatic choice:

**1. Prototyping**: When you're exploring an idea and types would slow you down

```typescript
// src/experiments/notification-ai.ts - Prototyping
// TODO: Add proper types once we decide on the API
export function generateNotificationMessage(context: any): string {
  // Experimenting with AI-generated messages
  // Will add proper types once we finalize the context structure
  return `Generated message based on ${context}`;
}
```

**2. Gradual migration**: When converting JavaScript to TypeScript

```typescript
// src/legacy/old-notification-system.ts - Gradual migration
// This file is being migrated from JavaScript
// Using any temporarily to get it compiling, will add types incrementally

export function legacyNotificationHandler(data: any): void {
  // TODO: Type this properly
  console.log('Legacy handler:', data);
}
```

**3. Truly dynamic data**: When the shape of data is genuinely unknowable

```typescript
// src/utils/debug.ts - Truly dynamic data
export function debugLog(label: string, data: any): void {
  // This is a debug utility that accepts literally anything
  // Using any is appropriate here
  console.log(`[${label}]`, data);
}
```

**4. Working around TypeScript limitations**: When TypeScript's type system can't express what you need

```typescript
// src/utils/deep-merge.ts - TypeScript limitation
export function deepMerge<T>(target: T, source: any): T {
  // Deep merge is genuinely hard to type correctly
  // The proper type would be incredibly complex
  // Using any here is pragmatic
  
  if (typeof target !== 'object' || typeof source !== 'object') {
    return source;
  }

  const result = { ...target };
  
  for (const key in source) {
    if (source.hasOwnProperty(key)) {
      if (typeof source[key] === 'object' && !Array.isArray(source[key])) {
        (result as any)[key] = deepMerge((target as any)[key] || {}, source[key]);
      } else {
        (result as any)[key] = source[key];
      }
    }
  }

  return result;
}
```

### The Rules of `any`

**Rule 1: Contain the blast radius**

When you use `any`, limit its scope. Don't let it leak into the rest of your codebase.

```typescript
// ❌ Bad: any leaks everywhere
export function processNotification(data: any) {
  return data;  // Returns any
}

// ✓ Good: any is contained
export function processNotification(data: any): NotificationProps {
  // Validate and transform to proper type
  return {
    variant: data.variant || 'info',
    message: data.message || 'No message',
    dismissible: Boolean(data.dismissible),
  };
}
```

**Rule 2: Document why you used `any`**

Always leave a comment explaining why `any` was necessary.

```typescript
// ✓ Good: Documented
export function trackEvent(event: string, properties: any): void {
  // Using any because the analytics library doesn't have types
  // and the property shape varies by event type
  analytics.track(event, properties);
}
```

**Rule 3: Prefer `unknown` when receiving external data**

Use `unknown` instead of `any` when you don't know the type yet but will validate it.

```typescript
// ❌ Bad: Using any for external data
function parseApiResponse(response: any): NotificationProps {
  return response;  // No validation
}

// ✓ Good: Using unknown with validation
function parseApiResponse(response: unknown): NotificationProps {
  if (!isValidNotification(response)) {
    throw new Error('Invalid notification data');
  }
  return response;
}

function isValidNotification(value: unknown): value is NotificationProps {
  // Validation logic
  return (
    typeof value === 'object' &&
    value !== null &&
    'variant' in value &&
    'message' in value
  );
}
```

**Rule 4: Use type assertions instead of `any` when possible**

If you know the type but TypeScript doesn't, use a type assertion instead of `any`.

```typescript
// ❌ Bad: Using any
const element = document.getElementById('root') as any;
element.style.color = 'red';

// ✓ Good: Using type assertion
const element = document.getElementById('root') as HTMLElement;
element.style.color = 'red';

// ✓ Better: Using type guard
const element = document.getElementById('root');
if (element instanceof HTMLElement) {
  element.style.color = 'red';
}
```

### When to Apply `any` vs. Alternatives

| Situation | Use | Why |
|-----------|-----|-----|
| External library without types | `any` at boundary | Pragmatic, contained |
| Receiving external data | `unknown` | Forces validation |
| Prototyping | `any` with TODO | Speed over safety |
| Gradual migration | `any` with TODO | Incremental improvement |
| Debug utilities | `any` | Genuinely accepts anything |
| TypeScript limitation | `any` with comment | Workaround documented |
| You know the type | Type assertion | More specific than `any` |
| DOM manipulation | Type guard or assertion | Safer than `any` |

### The Complete Journey: From Naive to Professional

Let's look at how our notification system evolved through this chapter:

| Iteration | Problem | Solution | Type Safety Impact |
|-----------|---------|----------|-------------------|
| 0 | Boolean flags everywhere | Naive approach | Invalid states compile |
| 1 | Invalid combinations possible | Discriminated unions | Invalid states unrepresentable |
| 2 | Adding new variants breaks code | Exhaustiveness checking | Compiler forces handling all cases |
| 3 | Need to extract specific variants | Extract/Exclude utilities | Type-safe filtering |
| 4 | Context used outside provider | undefined + runtime check | Clear error messages |
| 5 | Generic filtering logic | Generic hooks | Reusable, type-safe |
| 6 | Third-party library without types | Strategic `any` at boundary | Pragmatic compromise |
| 7 | External data validation | `unknown` + type guards | Safe external data handling |

### Final Implementation: Production-Ready Notification System

Here's our complete, production-ready notification system with all the TypeScript patterns we've learned:

```typescript
// src/types/notifications.ts - Final version
export type SuccessNotification = {
  variant: 'success';
  message: string;
  dismissible?: boolean;
};

export type ErrorNotification = {
  variant: 'error';
  message: string;
  dismissible?: boolean;
  action?: {
    label: string;
    onClick: () => void;
  };
};

export type WarningNotification = {
  variant: 'warning';
  message: string;
  dismissible?: boolean;
};

export type InfoNotification = {
  variant: 'info';
  message: string;
  dismissible?: boolean;
};

export type LoadingNotification = {
  variant: 'loading';
  message: string;
};

export type ConfirmationNotification = {
  variant: 'confirmation';
  message: string;
  confirmText: string;
  cancelText: string;
  onConfirm: () => void;
  onCancel: () => void;
  severity: 'warning' | 'danger';
};

export type NotificationProps = 
  | SuccessNotification 
  | ErrorNotification 
  | WarningNotification 
  | InfoNotification
  | LoadingNotification
  | ConfirmationNotification;

export type NotificationWithId = NotificationProps & { id: string };

// Utility types
export type ErrorNotificationOnly = Extract<NotificationProps, { variant: 'error' }>;
export type DismissibleNotifications = Extract<NotificationProps, { dismissible?: boolean }>;
export type InteractiveNotifications = Exclude<NotificationProps, { variant: 'loading' }>;

// Configuration
export type NotificationConfig = Record<
  NotificationProps['variant'],
  {
    icon: string;
    backgroundColor: string;
    textColor: string;
  }
>;

export const notificationConfig: NotificationConfig = {
  success: { icon: '✓', backgroundColor: 'bg-green-100', textColor: 'text-green-800' },
  error: { icon: '✗', backgroundColor: 'bg-red-100', textColor: 'text-red-800' },
  warning: { icon: '⚠', backgroundColor: 'bg-yellow-100', textColor: 'text-yellow-800' },
  info: { icon: 'ℹ', backgroundColor: 'bg-blue-100', textColor: 'text-blue-800' },
  loading: { icon: '⟳', backgroundColor: 'bg-gray-100', textColor: 'text-gray-800' },
  confirmation: { icon: '?', backgroundColor: 'bg-purple-100', textColor: 'text-purple-800' },
};
```

```tsx
// src/contexts/NotificationContext.tsx - Final version
'use client';

import { createContext, useContext, useState, useCallback, ReactNode } from 'react';
import type { NotificationProps, NotificationWithId } from '@/types/notifications';

type NotificationContextType = {
  notifications: NotificationWithId[];
  addNotification: (notification: NotificationProps) => string;
  removeNotification: (id: string) => void;
  clearAll: () => void;
};

const NotificationContext = createContext<NotificationContextType | undefined>(undefined);

export function NotificationProvider({ children }: { children: ReactNode }) {
  const [notifications, setNotifications] = useState<NotificationWithId[]>([]);

  const addNotification = useCallback((notification: NotificationProps): string => {
    const id = Math.random().toString(36).substr(2, 9);
    setNotifications((prev) => [...prev, { ...notification, id }]);
    return id;
  }, []);

  const removeNotification = useCallback((id: string) => {
    setNotifications((prev) => prev.filter((n) => n.id !== id));
  }, []);

  const clearAll = useCallback(() => {
    setNotifications([]);
  }, []);

  return (
    <NotificationContext.Provider 
      value={{ notifications, addNotification, removeNotification, clearAll }}
    >
      {children}
    </NotificationContext.Provider>
  );
}

export function useNotifications() {
  const context = useContext(NotificationContext);
  if (context === undefined) {
    throw new Error('useNotifications must be used within a NotificationProvider');
  }
  return context;
}
```

```tsx
// src/components/Notification.tsx - Final version
import type { NotificationProps } from '@/types/notifications';
import { notificationConfig } from '@/types/notifications';

export function Notification(props: NotificationProps) {
  const config = notificationConfig[props.variant];

  const assertNever = (value: never): never => {
    throw new Error(`Unhandled variant: ${value}`);
  };

  return (
    <div className={`p-4 rounded ${config.backgroundColor}`}>
      <div className="flex items-start gap-3">
        <span className={`text-xl ${config.textColor}`}>{config.icon}</span>
        <div className="flex-1">
          <p className={config.textColor}>{props.message}</p>
          
          {props.variant === 'error' && props.action && (
            <button
              onClick={props.action.onClick}
              className="mt-2 text-sm underline"
            >
              {props.action.label}
            </button>
          )}
          
          {props.variant === 'confirmation' && (
            <div className="mt-3 flex gap-2">
              <button
                onClick={props.onConfirm}
                className={`px-4 py-2 rounded text-white ${
                  props.severity === 'danger' 
                    ? 'bg-red-600 hover:bg-red-700' 
                    : 'bg-yellow-600 hover:bg-yellow-700'
                }`}
              >
                {props.confirmText}
              </button>
              <button
                onClick={props.onCancel}
                className="px-4 py-2 rounded bg-gray-200 hover:bg-gray-300"
              >
                {props.cancelText}
              </button>
            </div>
          )}
        </div>
        
        {props.variant !== 'confirmation' && 
         props.variant !== 'loading' && 
         props.dismissible && (
          <button className="text-gray-500">×</button>
        )}
      </div>
    </div>
  );
}
```

### Lessons Learned

**1. Make invalid states unrepresentable**
Use discriminated unions to ensure only valid combinations of properties can exist. TypeScript becomes your design validator.

**2. Utility types are your friends**
`Extract`, `Exclude`, `Pick`, `Omit`, `Partial`, `Required`, and `Record` solve 90% of type manipulation needs. Learn them well.

**3. Context needs runtime checks**
Always make context values potentially `undefined` and add runtime checks in the hook. Clear error messages save debugging time.

**4. Generics enable reusable, type-safe code**
Generic hooks and functions let you write code once and use it with multiple types, without sacrificing type safety.

**5. `any` is a tool, not a failure**
Use `any` strategically at boundaries with untyped code. Document why, contain the scope, and prefer `unknown` for external data.

**6. TypeScript is a design tool**
The type system isn't just for catching bugs—it's for designing better APIs. If your types are hard to use, your API probably is too.

**7. Exhaustiveness checking prevents bugs**
Use the `assertNever` pattern to ensure you handle all cases of a discriminated union. When you add a new variant, TypeScript will tell you everywhere you need to update.

This notification system demonstrates professional TypeScript patterns that scale to real-world applications. The types guide you toward correct usage, catch mistakes at compile time, and document the API for other developers.
