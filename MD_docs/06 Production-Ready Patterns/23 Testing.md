# Chapter 23: Testing

## Vitest: modern, fast testing

## The Problem Testing Solves

You've built a payment form. It works perfectly—you've clicked through it dozens of times. You ship it to production. Two weeks later, you add a discount code feature. You test the discount code. It works. You ship it.

The next day, your support team reports that users can't submit payments anymore. The submit button is disabled even when the form is valid. You investigate and discover that your discount code logic accidentally broke the form validation. The bug was there for 12 hours before anyone noticed.

**This is the failure manual testing doesn't catch: regressions.**

Every time you change code, you risk breaking something that used to work. Manual testing can't scale—you can't click through every feature after every change. Automated tests catch regressions before they reach production.

## Reference Implementation: Payment Form Component

We'll build a realistic payment form that accepts credit card information, validates input, and handles submission. This component will evolve through the chapter as we add tests that catch real bugs.

**Project Structure**:
```
src/
├── components/
│   ├── PaymentForm.tsx          ← Our reference implementation
│   ├── PaymentForm.test.tsx     ← Tests we'll build
│   └── CreditCardInput.tsx      ← Child component
├── lib/
│   ├── validation.ts            ← Validation logic
│   └── validation.test.ts       ← Unit tests
└── __tests__/
    └── setup.ts                 ← Test configuration
```

### Initial Implementation: The Untested Form

Here's our starting point—a payment form that works but has no tests:

```tsx
// src/components/PaymentForm.tsx
import { useState } from 'react';

interface PaymentFormProps {
  onSubmit: (data: PaymentData) => Promise<void>;
  amount: number;
}

interface PaymentData {
  cardNumber: string;
  expiryDate: string;
  cvv: string;
  cardholderName: string;
}

export function PaymentForm({ onSubmit, amount }: PaymentFormProps) {
  const [cardNumber, setCardNumber] = useState('');
  const [expiryDate, setExpiryDate] = useState('');
  const [cvv, setCvv] = useState('');
  const [cardholderName, setCardholderName] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const isValid = 
    cardNumber.length === 16 &&
    expiryDate.length === 5 &&
    cvv.length === 3 &&
    cardholderName.length > 0;

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!isValid) return;

    setIsSubmitting(true);
    setError(null);

    try {
      await onSubmit({
        cardNumber,
        expiryDate,
        cvv,
        cardholderName,
      });
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Payment failed');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="payment-form">
      <h2>Payment Details</h2>
      <p className="amount">Amount: ${amount.toFixed(2)}</p>

      <div className="form-group">
        <label htmlFor="cardNumber">Card Number</label>
        <input
          id="cardNumber"
          type="text"
          value={cardNumber}
          onChange={(e) => setCardNumber(e.target.value)}
          placeholder="1234567812345678"
          maxLength={16}
        />
      </div>

      <div className="form-row">
        <div className="form-group">
          <label htmlFor="expiryDate">Expiry Date</label>
          <input
            id="expiryDate"
            type="text"
            value={expiryDate}
            onChange={(e) => setExpiryDate(e.target.value)}
            placeholder="MM/YY"
            maxLength={5}
          />
        </div>

        <div className="form-group">
          <label htmlFor="cvv">CVV</label>
          <input
            id="cvv"
            type="text"
            value={cvv}
            onChange={(e) => setCvv(e.target.value)}
            placeholder="123"
            maxLength={3}
          />
        </div>
      </div>

      <div className="form-group">
        <label htmlFor="cardholderName">Cardholder Name</label>
        <input
          id="cardholderName"
          type="text"
          value={cardholderName}
          onChange={(e) => setCardholderName(e.target.value)}
          placeholder="John Doe"
        />
      </div>

      {error && (
        <div className="error" role="alert">
          {error}
        </div>
      )}

      <button
        type="submit"
        disabled={!isValid || isSubmitting}
      >
        {isSubmitting ? 'Processing...' : `Pay $${amount.toFixed(2)}`}
      </button>
    </form>
  );
}
```

This form works. You can fill it out, submit it, see loading states, and handle errors. But how do you know it will keep working after you make changes?

## Setting Up Vitest

Vitest is a modern test runner built for Vite projects. It's fast, has excellent TypeScript support, and provides a Jest-compatible API that most developers already know.

### Installation

First, install Vitest and testing utilities:

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

**What each package does**:
- `vitest` - The test runner itself
- `@testing-library/react` - Utilities for testing React components
- `@testing-library/jest-dom` - Custom matchers for DOM assertions
- `@testing-library/user-event` - Simulates real user interactions
- `jsdom` - Simulates a browser environment in Node.js

### Configuration

Add Vitest configuration to your `vite.config.ts`:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/__tests__/setup.ts',
    css: true,
  },
});
```

**Configuration explained**:
- `globals: true` - Makes `describe`, `it`, `expect` available without imports
- `environment: 'jsdom'` - Simulates browser APIs (DOM, window, etc.)
- `setupFiles` - Runs before each test file
- `css: true` - Processes CSS imports (needed for styled components)

### Test Setup File

Create the setup file that runs before all tests:

```typescript
// src/__tests__/setup.ts
import { expect, afterEach } from 'vitest';
import { cleanup } from '@testing-library/react';
import * as matchers from '@testing-library/jest-dom/matchers';

// Extend Vitest's expect with jest-dom matchers
expect.extend(matchers);

// Cleanup after each test
afterEach(() => {
  cleanup();
});
```

This setup:
1. Adds custom matchers like `toBeInTheDocument()`, `toHaveValue()`, etc.
2. Automatically cleans up rendered components after each test
3. Prevents test pollution (one test affecting another)

### Package.json Scripts

Add test scripts to `package.json`:

```json
{
  "scripts": {
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest --coverage"
  }
}
```

**Script purposes**:
- `test` - Runs tests in watch mode (re-runs on file changes)
- `test:ui` - Opens a browser UI for exploring tests
- `test:coverage` - Generates code coverage report

## Your First Test: Does It Render?

The simplest test verifies that the component renders without crashing:

```tsx
// src/components/PaymentForm.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import { PaymentForm } from './PaymentForm';

describe('PaymentForm', () => {
  it('renders the form with all fields', () => {
    const mockOnSubmit = vi.fn();
    
    render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);

    // Check that key elements are present
    expect(screen.getByText('Payment Details')).toBeInTheDocument();
    expect(screen.getByLabelText('Card Number')).toBeInTheDocument();
    expect(screen.getByLabelText('Expiry Date')).toBeInTheDocument();
    expect(screen.getByLabelText('CVV')).toBeInTheDocument();
    expect(screen.getByLabelText('Cardholder Name')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /pay \$99\.99/i })).toBeInTheDocument();
  });
});
```

**What this test does**:
1. Creates a mock function for `onSubmit` using `vi.fn()`
2. Renders the component with test props
3. Queries the DOM for expected elements
4. Asserts they exist using `toBeInTheDocument()`

**Run the test**:

```bash
npm test
```

**Terminal Output**:
```
 ✓ src/components/PaymentForm.test.tsx (1)
   ✓ PaymentForm (1)
     ✓ renders the form with all fields

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  10:23:45
   Duration  234ms
```

**Success!** The test passes. But this test is shallow—it only checks that elements exist. It doesn't verify behavior.

## The Anatomy of a Vitest Test

Let's break down the structure:

```typescript
describe('ComponentName', () => {
  // Test suite - groups related tests

  it('describes what the test verifies', () => {
    // Individual test case
    
    // 1. Arrange - Set up test data and render component
    const mockFn = vi.fn();
    render(<Component prop={mockFn} />);

    // 2. Act - Perform user actions
    const button = screen.getByRole('button');
    await userEvent.click(button);

    // 3. Assert - Verify expected outcomes
    expect(mockFn).toHaveBeenCalled();
  });
});
```

**The AAA Pattern**:
- **Arrange** - Set up the test scenario
- **Act** - Trigger the behavior you're testing
- **Assert** - Verify the outcome

This pattern makes tests readable and maintainable.

## Vitest's Key Features

### Fast Execution

Vitest runs tests in parallel and only re-runs tests affected by code changes:

```bash
# Watch mode automatically re-runs tests on file changes
npm test

# Run tests once (CI mode)
npm test -- --run
```

**Terminal Output (Watch Mode)**:
```
 RERUN  src/components/PaymentForm.test.tsx

 ✓ src/components/PaymentForm.test.tsx (1) 89ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  10:25:12
   Duration  89ms (in thread 45ms, 197.78%)

 PASS  Waiting for file changes...
       press h to show help, press q to quit
```

### Mocking with vi

Vitest provides `vi` for creating mocks, spies, and stubs:

```typescript
import { vi } from 'vitest';

// Mock a function
const mockFn = vi.fn();
mockFn('test');
expect(mockFn).toHaveBeenCalledWith('test');

// Mock a function with return value
const mockFetch = vi.fn().mockResolvedValue({ ok: true, json: async () => ({ data: 'test' }) });

// Spy on an existing function
const spy = vi.spyOn(console, 'log');
console.log('test');
expect(spy).toHaveBeenCalledWith('test');
spy.mockRestore();

// Mock a module
vi.mock('./api', () => ({
  fetchUser: vi.fn().mockResolvedValue({ id: 1, name: 'Test User' }),
}));
```

### Snapshot Testing

Vitest supports snapshot testing for catching unintended UI changes:

```typescript
it('matches snapshot', () => {
  const { container } = render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);
  expect(container).toMatchSnapshot();
});
```

**When to use snapshots**:
- ✅ Testing component structure that shouldn't change often
- ✅ Catching unintended layout changes
- ❌ Testing dynamic content (dates, IDs, random values)
- ❌ As a substitute for meaningful assertions

### Coverage Reports

Generate coverage to see what code is tested:

```bash
npm test -- --coverage
```

**Terminal Output**:
```
 % Coverage report from v8
--------------------|---------|----------|---------|---------|-------------------
File                | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
--------------------|---------|----------|---------|---------|-------------------
All files           |   45.45 |       50 |      40 |   45.45 |
 PaymentForm.tsx    |   45.45 |       50 |      40 |   45.45 | 28-45
--------------------|---------|----------|---------|---------|-------------------
```

**What this tells you**:
- Only 45% of statements are executed by tests
- Lines 28-45 (the submit handler) are untested
- We need more tests to cover error handling and submission

## Why Vitest Over Jest?

**Vitest advantages**:
- **Faster** - Native ESM support, no transpilation needed
- **Better DX** - Instant HMR for tests, built-in TypeScript support
- **Vite integration** - Uses your existing Vite config
- **Modern** - Designed for 2025, not 2015

**When to use Jest instead**:
- Legacy projects already using Jest
- Need specific Jest plugins not available for Vitest
- Team familiarity outweighs technical benefits

For new projects in 2025, Vitest is the pragmatic choice.

## React Testing Library: user-centric tests

## The Failure: Testing Implementation Details

Let's write a test the wrong way first. Many developers test React components like this:

```tsx
// ❌ BAD: Testing implementation details
it('updates state when card number changes', () => {
  const { result } = renderHook(() => useState(''));
  const [cardNumber, setCardNumber] = result.current;
  
  act(() => {
    setCardNumber('1234567812345678');
  });
  
  expect(result.current[0]).toBe('1234567812345678');
});
```

**Run this test**:

**Terminal Output**:
```
 ✓ src/components/PaymentForm.test.tsx (2)
   ✓ PaymentForm (2)
     ✓ renders the form with all fields
     ✓ updates state when card number changes

 Test Files  1 passed (1)
      Tests  2 passed (2)
```

**The test passes. So what's wrong?**

### Diagnostic Analysis: Why This Test Is Useless

**The problem**: This test verifies that `useState` works. But `useState` is React's code, not yours. You're testing the framework, not your component.

**What happens when you refactor**:

Imagine you refactor to use `useReducer` instead of `useState`:

```tsx
// Refactored to useReducer
const [state, dispatch] = useReducer(reducer, initialState);

// Your test breaks even though the component still works
// The test was coupled to implementation details
```

**The test breaks** even though the component behavior is identical from the user's perspective. The user doesn't care whether you use `useState` or `useReducer`—they only care that typing in the input updates the value.

**This is the fundamental problem with testing implementation details**: Tests break when you refactor, even when behavior doesn't change.

## React Testing Library Philosophy

React Testing Library enforces a simple principle:

> **Test your components the way users interact with them.**

Users don't call `setState`. Users don't access component internals. Users:
- See text on the screen
- Click buttons
- Type in inputs
- Read error messages

Your tests should do the same.

## Iteration 1: Testing User Interactions

Let's rewrite our test to focus on user behavior:

```tsx
// src/components/PaymentForm.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { PaymentForm } from './PaymentForm';

describe('PaymentForm', () => {
  it('enables submit button when all fields are valid', async () => {
    const user = userEvent.setup();
    const mockOnSubmit = vi.fn();
    
    render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);

    // Initially, button should be disabled
    const submitButton = screen.getByRole('button', { name: /pay \$99\.99/i });
    expect(submitButton).toBeDisabled();

    // Fill out the form as a user would
    await user.type(screen.getByLabelText('Card Number'), '1234567812345678');
    await user.type(screen.getByLabelText('Expiry Date'), '12/25');
    await user.type(screen.getByLabelText('CVV'), '123');
    await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');

    // Now button should be enabled
    expect(submitButton).toBeEnabled();
  });
});
```

**Run the test**:

**Terminal Output**:
```
 ✓ src/components/PaymentForm.test.tsx (1)
   ✓ PaymentForm (1)
     ✓ enables submit button when all fields are valid

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  10:45:23
   Duration  312ms
```

**What changed**:
1. We use `userEvent` to simulate real typing
2. We query elements by their accessible labels (what users see)
3. We verify the button state (what users experience)
4. We never touch component internals

**This test is resilient**: You can refactor the component's state management, and this test will still pass as long as the user experience remains the same.

## The React Testing Library Query Priority

React Testing Library provides multiple ways to query elements. Use them in this order:

### 1. Queries Accessible to Everyone

**Prefer these** - they reflect how users and assistive technologies interact:

```typescript
// ✅ BEST: By role (most accessible)
screen.getByRole('button', { name: /submit/i });
screen.getByRole('textbox', { name: /email/i });

// ✅ GOOD: By label text (what users see)
screen.getByLabelText('Email Address');

// ✅ GOOD: By placeholder text
screen.getByPlaceholderText('Enter your email');

// ✅ GOOD: By text content
screen.getByText('Welcome back!');
```

### 2. Semantic Queries

**Use when accessible queries don't work**:

```typescript
// ⚠️ OK: By alt text (for images)
screen.getByAltText('User avatar');

// ⚠️ OK: By title attribute
screen.getByTitle('Close dialog');
```

### 3. Test IDs (Last Resort)

**Only when nothing else works**:

```typescript
// ❌ AVOID: By test ID (not user-facing)
screen.getByTestId('submit-button');

// In component:
<button data-testid="submit-button">Submit</button>
```

**Why avoid test IDs?**
- They don't reflect user experience
- They add noise to your markup
- They can hide accessibility issues

If you need a test ID, it often means your component lacks proper semantic HTML or ARIA attributes.

## Query Variants: get, query, find

React Testing Library provides three query variants:

### getBy - Throws if not found

```typescript
// Throws error immediately if element doesn't exist
const button = screen.getByRole('button');
// Use when: Element should definitely be present
```

### queryBy - Returns null if not found

```typescript
// Returns null if element doesn't exist
const error = screen.queryByRole('alert');
expect(error).not.toBeInTheDocument();
// Use when: Testing that element is NOT present
```

### findBy - Waits for element to appear

```typescript
// Waits up to 1000ms for element to appear
const message = await screen.findByText('Payment successful');
// Use when: Element appears asynchronously
```

## Iteration 2: Testing Form Submission

Now let's test the actual submission flow:

```tsx
// src/components/PaymentForm.test.tsx
describe('PaymentForm', () => {
  it('calls onSubmit with form data when submitted', async () => {
    const user = userEvent.setup();
    const mockOnSubmit = vi.fn().mockResolvedValue(undefined);
    
    render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);

    // Fill out the form
    await user.type(screen.getByLabelText('Card Number'), '1234567812345678');
    await user.type(screen.getByLabelText('Expiry Date'), '12/25');
    await user.type(screen.getByLabelText('CVV'), '123');
    await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');

    // Submit the form
    await user.click(screen.getByRole('button', { name: /pay \$99\.99/i }));

    // Verify onSubmit was called with correct data
    expect(mockOnSubmit).toHaveBeenCalledTimes(1);
    expect(mockOnSubmit).toHaveBeenCalledWith({
      cardNumber: '1234567812345678',
      expiryDate: '12/25',
      cvv: '123',
      cardholderName: 'John Doe',
    });
  });
});
```

**Run the test**:

**Terminal Output**:
```
 ✓ src/components/PaymentForm.test.tsx (2)
   ✓ PaymentForm (2)
     ✓ enables submit button when all fields are valid
     ✓ calls onSubmit with form data when submitted

 Test Files  1 passed (1)
      Tests  2 passed (2)
   Start at  10:52:18
   Duration  445ms
```

**What we verified**:
1. User can fill out all fields
2. User can click submit button
3. Component calls `onSubmit` with correct data
4. Component calls `onSubmit` exactly once (no double submission)

## The Failure: Async State Updates

Let's test the loading state during submission:

```tsx
it('shows loading state during submission', async () => {
  const user = userEvent.setup();
  const mockOnSubmit = vi.fn().mockImplementation(
    () => new Promise(resolve => setTimeout(resolve, 100))
  );
  
  render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);

  // Fill and submit form
  await user.type(screen.getByLabelText('Card Number'), '1234567812345678');
  await user.type(screen.getByLabelText('Expiry Date'), '12/25');
  await user.type(screen.getByLabelText('CVV'), '123');
  await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
  await user.click(screen.getByRole('button', { name: /pay \$99\.99/i }));

  // Check for loading state
  expect(screen.getByRole('button', { name: /processing/i })).toBeInTheDocument();
});
```

**Run the test**:

**Terminal Output**:
```
 FAIL  src/components/PaymentForm.test.tsx > PaymentForm > shows loading state during submission
TestingLibraryElementError: Unable to find an accessible element with the role "button" and name `/processing/i`

Here are the accessible roles:

  button:

  Name "Pay $99.99":
  <button
    disabled=""
    type="submit"
  />
```

### Diagnostic Analysis: Race Condition in Test

**What happened**:
1. We clicked the submit button
2. The component started the async `onSubmit` call
3. We immediately queried for the "Processing..." button
4. But React hadn't re-rendered yet with the loading state
5. The test found the old "Pay $99.99" button instead

**The problem**: We're testing async behavior synchronously.

**The solution**: Use `findBy` to wait for the loading state:

```tsx
it('shows loading state during submission', async () => {
  const user = userEvent.setup();
  const mockOnSubmit = vi.fn().mockImplementation(
    () => new Promise(resolve => setTimeout(resolve, 100))
  );
  
  render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);

  // Fill and submit form
  await user.type(screen.getByLabelText('Card Number'), '1234567812345678');
  await user.type(screen.getByLabelText('Expiry Date'), '12/25');
  await user.type(screen.getByLabelText('CVV'), '123');
  await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
  await user.click(screen.getByRole('button', { name: /pay \$99\.99/i }));

  // Wait for loading state to appear
  expect(await screen.findByRole('button', { name: /processing/i })).toBeInTheDocument();
  
  // Wait for loading state to disappear
  await waitFor(() => {
    expect(screen.queryByRole('button', { name: /processing/i })).not.toBeInTheDocument();
  });
});
```

**Run the test**:

**Terminal Output**:
```
 ✓ src/components/PaymentForm.test.tsx (3)
   ✓ PaymentForm (3)
     ✓ enables submit button when all fields are valid
     ✓ calls onSubmit with form data when submitted
     ✓ shows loading state during submission

 Test Files  1 passed (1)
      Tests  3 passed (3)
   Start at  11:05:42
   Duration  612ms
```

**Success!** The test now properly waits for async state updates.

## Iteration 3: Testing Error Handling

Let's test what happens when submission fails:

```tsx
it('displays error message when submission fails', async () => {
  const user = userEvent.setup();
  const mockOnSubmit = vi.fn().mockRejectedValue(
    new Error('Payment declined')
  );
  
  render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);

  // Fill and submit form
  await user.type(screen.getByLabelText('Card Number'), '1234567812345678');
  await user.type(screen.getByLabelText('Expiry Date'), '12/25');
  await user.type(screen.getByLabelText('CVV'), '123');
  await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
  await user.click(screen.getByRole('button', { name: /pay \$99\.99/i }));

  // Wait for error message to appear
  const errorMessage = await screen.findByRole('alert');
  expect(errorMessage).toHaveTextContent('Payment declined');

  // Button should be enabled again for retry
  expect(screen.getByRole('button', { name: /pay \$99\.99/i })).toBeEnabled();
});
```

**Run the test**:

**Terminal Output**:
```
 ✓ src/components/PaymentForm.test.tsx (4)
   ✓ PaymentForm (4)
     ✓ enables submit button when all fields are valid
     ✓ calls onSubmit with form data when submitted
     ✓ shows loading state during submission
     ✓ displays error message when submission fails

 Test Files  1 passed (1)
      Tests  4 passed (4)
   Start at  11:12:35
   Duration  723ms
```

**What we verified**:
1. Error message appears with correct text
2. Error has `role="alert"` for accessibility
3. Button re-enables after error (user can retry)

## Testing Accessibility

React Testing Library encourages accessible components by making inaccessible components hard to test:

```tsx
it('has accessible form labels', () => {
  render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);

  // These queries will fail if labels aren't properly associated
  expect(screen.getByLabelText('Card Number')).toBeInTheDocument();
  expect(screen.getByLabelText('Expiry Date')).toBeInTheDocument();
  expect(screen.getByLabelText('CVV')).toBeInTheDocument();
  expect(screen.getByLabelText('Cardholder Name')).toBeInTheDocument();
});

it('has accessible button text', () => {
  render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);

  // Button text should be descriptive
  const button = screen.getByRole('button', { name: /pay \$99\.99/i });
  expect(button).toBeInTheDocument();
});

it('announces errors to screen readers', async () => {
  const user = userEvent.setup();
  const mockOnSubmit = vi.fn().mockRejectedValue(new Error('Payment declined'));
  
  render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);

  await user.type(screen.getByLabelText('Card Number'), '1234567812345678');
  await user.type(screen.getByLabelText('Expiry Date'), '12/25');
  await user.type(screen.getByLabelText('CVV'), '123');
  await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
  await user.click(screen.getByRole('button', { name: /pay \$99\.99/i }));

  // Error should have role="alert" for screen reader announcement
  const error = await screen.findByRole('alert');
  expect(error).toHaveTextContent('Payment declined');
});
```

**If these tests fail**, it means your component has accessibility issues. Fix the component, not the test.

## Common Testing Library Patterns

### Testing Conditional Rendering

```tsx
it('hides error message initially', () => {
  render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);
  
  // Use queryBy when testing absence
  expect(screen.queryByRole('alert')).not.toBeInTheDocument();
});
```

### Testing Input Validation

```tsx
it('keeps submit button disabled with invalid card number', async () => {
  const user = userEvent.setup();
  render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);

  // Fill form with invalid card number (too short)
  await user.type(screen.getByLabelText('Card Number'), '12345');
  await user.type(screen.getByLabelText('Expiry Date'), '12/25');
  await user.type(screen.getByLabelText('CVV'), '123');
  await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');

  // Button should remain disabled
  expect(screen.getByRole('button', { name: /pay/i })).toBeDisabled();
});
```

### Testing Focus Management

```tsx
it('focuses first input on mount', () => {
  render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);
  
  const cardNumberInput = screen.getByLabelText('Card Number');
  expect(cardNumberInput).toHaveFocus();
});
```

### Testing Keyboard Navigation

```tsx
it('submits form on Enter key', async () => {
  const user = userEvent.setup();
  const mockOnSubmit = vi.fn().mockResolvedValue(undefined);
  
  render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);

  // Fill form
  await user.type(screen.getByLabelText('Card Number'), '1234567812345678');
  await user.type(screen.getByLabelText('Expiry Date'), '12/25');
  await user.type(screen.getByLabelText('CVV'), '123');
  await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
  
  // Press Enter in last field
  await user.keyboard('{Enter}');

  expect(mockOnSubmit).toHaveBeenCalled();
});
```

## When to Apply: Testing Library Decision Framework

**Use React Testing Library when**:
- Testing user-facing behavior
- Verifying accessibility
- Testing component integration
- Building confidence in user experience

**Don't use React Testing Library for**:
- Testing pure functions (use plain Vitest)
- Testing implementation details
- Testing third-party libraries
- Performance benchmarking

**The rule**: If a user can't do it, don't test it. If a user can do it, test it.

## What to test (and what to skip)

## The Failure: Testing Everything

You're excited about testing. You write tests for everything:

```tsx
// ❌ Testing React internals
it('useState returns array with two elements', () => {
  const [state, setState] = useState(0);
  expect(Array.isArray([state, setState])).toBe(true);
  expect([state, setState]).toHaveLength(2);
});

// ❌ Testing third-party libraries
it('React Router navigates correctly', () => {
  const navigate = useNavigate();
  navigate('/test');
  expect(window.location.pathname).toBe('/test');
});

// ❌ Testing CSS
it('button has correct background color', () => {
  render(<Button />);
  const button = screen.getByRole('button');
  expect(button).toHaveStyle({ backgroundColor: 'blue' });
});

// ❌ Testing implementation details
it('component uses useEffect', () => {
  const spy = vi.spyOn(React, 'useEffect');
  render(<Component />);
  expect(spy).toHaveBeenCalled();
});
```

**Run these tests**:

**Terminal Output**:
```
 ✓ src/components/BadTests.test.tsx (4)
   ✓ useState returns array with two elements
   ✓ React Router navigates correctly
   ✓ button has correct background color
   ✓ component uses useEffect

 Test Files  1 passed (1)
      Tests  4 passed (4)
   Start at  11:45:23
   Duration  234ms
```

**All tests pass. But they're all useless.**

### Diagnostic Analysis: Why These Tests Waste Time

**Problem 1: Testing React internals**
- You're testing that React works, not that your code works
- React is already tested by the React team
- These tests add no value

**Problem 2: Testing third-party libraries**
- React Router is tested by its maintainers
- Your tests duplicate their work
- If React Router breaks, their tests will catch it

**Problem 3: Testing CSS**
- CSS is visual, not logical
- Tests can't verify that something "looks good"
- Visual regression testing tools (Percy, Chromatic) are better for this

**Problem 4: Testing implementation details**
- Tests break when you refactor
- Tests don't verify user experience
- Tests become maintenance burden

**The cost**: You spend time writing and maintaining tests that provide no value. When you refactor, these tests break even though behavior is unchanged. You lose trust in your test suite.

## The Testing Pyramid

Not all tests are created equal. The testing pyramid shows the ideal distribution:

```
        /\
       /  \
      / E2E \         ← Few, slow, expensive
     /--------\
    /          \
   / Integration \    ← Some, moderate speed
  /--------------\
 /                \
/   Unit Tests     \  ← Many, fast, cheap
--------------------
```

**Unit Tests (70%)**:
- Test individual functions and components in isolation
- Fast to run (milliseconds)
- Easy to write and maintain
- Provide specific failure messages

**Integration Tests (20%)**:
- Test how multiple components work together
- Moderate speed (seconds)
- More realistic than unit tests
- Catch integration bugs

**E2E Tests (10%)**:
- Test complete user flows in real browser
- Slow to run (minutes)
- Expensive to maintain
- Catch bugs that only appear in production environment

## What to Test: The Decision Framework

### ✅ Test User-Facing Behavior

**Test what users can see and do**:

```tsx
// ✅ GOOD: User can submit form
it('submits payment when form is valid', async () => {
  const user = userEvent.setup();
  const mockOnSubmit = vi.fn().mockResolvedValue(undefined);
  
  render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);
  
  await user.type(screen.getByLabelText('Card Number'), '1234567812345678');
  await user.type(screen.getByLabelText('Expiry Date'), '12/25');
  await user.type(screen.getByLabelText('CVV'), '123');
  await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
  await user.click(screen.getByRole('button', { name: /pay/i }));

  expect(mockOnSubmit).toHaveBeenCalled();
});
```

### ✅ Test Edge Cases and Error States

**Test what happens when things go wrong**:

```tsx
// ✅ GOOD: User sees error message on failure
it('displays error when payment fails', async () => {
  const user = userEvent.setup();
  const mockOnSubmit = vi.fn().mockRejectedValue(new Error('Insufficient funds'));
  
  render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);
  
  // Fill and submit form
  await user.type(screen.getByLabelText('Card Number'), '1234567812345678');
  await user.type(screen.getByLabelText('Expiry Date'), '12/25');
  await user.type(screen.getByLabelText('CVV'), '123');
  await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
  await user.click(screen.getByRole('button', { name: /pay/i }));

  expect(await screen.findByRole('alert')).toHaveTextContent('Insufficient funds');
});
```

### ✅ Test Accessibility

**Test that assistive technologies work**:

```tsx
// ✅ GOOD: Screen readers can navigate form
it('has accessible form labels', () => {
  render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);
  
  expect(screen.getByLabelText('Card Number')).toBeInTheDocument();
  expect(screen.getByLabelText('Expiry Date')).toBeInTheDocument();
  expect(screen.getByLabelText('CVV')).toBeInTheDocument();
});
```

### ✅ Test Business Logic

**Test pure functions that contain logic**:

```typescript
// src/lib/validation.ts
export function validateCardNumber(cardNumber: string): boolean {
  // Remove spaces and dashes
  const cleaned = cardNumber.replace(/[\s-]/g, '');
  
  // Check length
  if (cleaned.length !== 16) return false;
  
  // Check if all digits
  if (!/^\d+$/.test(cleaned)) return false;
  
  // Luhn algorithm
  let sum = 0;
  let isEven = false;
  
  for (let i = cleaned.length - 1; i >= 0; i--) {
    let digit = parseInt(cleaned[i], 10);
    
    if (isEven) {
      digit *= 2;
      if (digit > 9) digit -= 9;
    }
    
    sum += digit;
    isEven = !isEven;
  }
  
  return sum % 10 === 0;
}
```

```typescript
// src/lib/validation.test.ts
import { describe, it, expect } from 'vitest';
import { validateCardNumber } from './validation';

describe('validateCardNumber', () => {
  it('accepts valid card numbers', () => {
    expect(validateCardNumber('4532015112830366')).toBe(true);
    expect(validateCardNumber('6011514433546201')).toBe(true);
  });

  it('rejects invalid card numbers', () => {
    expect(validateCardNumber('1234567812345678')).toBe(false);
    expect(validateCardNumber('4532015112830367')).toBe(false); // Wrong checksum
  });

  it('handles card numbers with spaces', () => {
    expect(validateCardNumber('4532 0151 1283 0366')).toBe(true);
  });

  it('handles card numbers with dashes', () => {
    expect(validateCardNumber('4532-0151-1283-0366')).toBe(true);
  });

  it('rejects non-numeric characters', () => {
    expect(validateCardNumber('4532015112830abc')).toBe(false);
  });

  it('rejects wrong length', () => {
    expect(validateCardNumber('453201511283036')).toBe(false); // 15 digits
    expect(validateCardNumber('45320151128303666')).toBe(false); // 17 digits
  });
});
```

**Why test this function**:
- Contains complex logic (Luhn algorithm)
- Has multiple edge cases
- Pure function (no side effects)
- Easy to test thoroughly

### ❌ Don't Test Implementation Details

**Don't test how the component works internally**:

```tsx
// ❌ BAD: Testing state variable names
it('has cardNumber state', () => {
  const { result } = renderHook(() => useState(''));
  expect(result.current[0]).toBe('');
});

// ❌ BAD: Testing that useEffect was called
it('uses useEffect', () => {
  const spy = vi.spyOn(React, 'useEffect');
  render(<Component />);
  expect(spy).toHaveBeenCalled();
});

// ❌ BAD: Testing component structure
it('renders a div with class name', () => {
  const { container } = render(<Component />);
  expect(container.querySelector('.payment-form')).toBeInTheDocument();
});
```

### ❌ Don't Test Third-Party Libraries

**Don't test code you didn't write**:

```tsx
// ❌ BAD: Testing React Router
it('useNavigate returns function', () => {
  const navigate = useNavigate();
  expect(typeof navigate).toBe('function');
});

// ❌ BAD: Testing React Query
it('useQuery returns data', () => {
  const { data } = useQuery(['key'], fetchFn);
  expect(data).toBeDefined();
});
```

### ❌ Don't Test Trivial Code

**Don't test code that can't reasonably break**:

```tsx
// ❌ BAD: Testing that props are passed
it('passes amount prop to child', () => {
  render(<PaymentForm amount={99.99} onSubmit={vi.fn()} />);
  expect(screen.getByText('$99.99')).toBeInTheDocument();
});

// ❌ BAD: Testing constant values
it('has correct title', () => {
  render(<PaymentForm amount={99.99} onSubmit={vi.fn()} />);
  expect(screen.getByText('Payment Details')).toBeInTheDocument();
});
```

### ❌ Don't Test Styles

**Don't test CSS unless it affects functionality**:

```tsx
// ❌ BAD: Testing CSS classes
it('has correct class name', () => {
  const { container } = render(<Button />);
  expect(container.firstChild).toHaveClass('btn-primary');
});

// ❌ BAD: Testing computed styles
it('button is blue', () => {
  render(<Button />);
  const button = screen.getByRole('button');
  expect(button).toHaveStyle({ backgroundColor: 'blue' });
});
```

**Exception**: Test styles that affect functionality:

```tsx
// ✅ OK: Testing visibility (affects functionality)
it('hides error message initially', () => {
  render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);
  expect(screen.queryByRole('alert')).not.toBeInTheDocument();
});

// ✅ OK: Testing disabled state (affects functionality)
it('disables submit button when form is invalid', () => {
  render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);
  expect(screen.getByRole('button')).toBeDisabled();
});
```

## The 80/20 Rule of Testing

**Focus on the 20% of tests that catch 80% of bugs**:

### High-Value Tests (Write These)

1. **Happy path**: User completes primary task successfully
2. **Error states**: User sees helpful error messages
3. **Edge cases**: Boundary conditions and unusual inputs
4. **Accessibility**: Screen readers and keyboard navigation work
5. **Business logic**: Complex calculations and validations

### Low-Value Tests (Skip These)

1. **Framework behavior**: Testing that React/libraries work
2. **Trivial code**: Simple prop passing and constant values
3. **Implementation details**: Internal state and lifecycle methods
4. **Styles**: CSS classes and computed styles
5. **Third-party code**: Libraries you didn't write

## Coverage Targets: The Pragmatic Approach

**Don't aim for 100% coverage**. Aim for high coverage of high-value code.

**Realistic targets**:
- **Business logic**: 90-100% coverage
- **UI components**: 60-80% coverage
- **Integration points**: 70-90% coverage
- **Overall project**: 70-80% coverage

**Why not 100%?**
- Diminishing returns after 80%
- Last 20% is often trivial code
- Time better spent on other quality measures

**Check coverage**:

```bash
npm test -- --coverage
```

**Terminal Output**:
```
 % Coverage report from v8
--------------------|---------|----------|---------|---------|-------------------
File                | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
--------------------|---------|----------|---------|---------|-------------------
All files           |   78.26 |    75.00 |   80.00 |   78.26 |
 PaymentForm.tsx    |   85.71 |    83.33 |   100.00|   85.71 | 45-48
 validation.ts      |   95.00 |    90.00 |   100.00|   95.00 | 67
--------------------|---------|----------|---------|---------|-------------------
```

**What to do with this**:
- ✅ 85% coverage on PaymentForm is good
- ✅ 95% coverage on validation is excellent
- ⚠️ Lines 45-48 uncovered - check if they're important
- ❌ Don't write tests just to hit 100%

## When to Apply: Testing Decision Tree

**Before writing a test, ask**:

1. **Can a user do this?**
   - Yes → Write the test
   - No → Don't test it

2. **Does this contain business logic?**
   - Yes → Write the test
   - No → Consider skipping

3. **Is this code I wrote?**
   - Yes → Consider testing
   - No → Don't test it

4. **Will this test catch real bugs?**
   - Yes → Write the test
   - No → Skip it

5. **Will this test break when I refactor?**
   - Yes → Reconsider the test
   - No → Write the test

**The golden rule**: Test behavior, not implementation. Test outcomes, not mechanisms.

## Integration tests that matter

## The Failure: Unit Tests Miss Integration Bugs

You have excellent unit test coverage. Every component works perfectly in isolation:

```tsx
// ✅ All unit tests pass
describe('PaymentForm', () => {
  it('submits payment data', async () => {
    const mockOnSubmit = vi.fn().mockResolvedValue(undefined);
    render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);
    // ... test passes
  });
});

describe('PaymentConfirmation', () => {
  it('displays confirmation message', () => {
    render(<PaymentConfirmation amount={99.99} />);
    // ... test passes
  });
});
```

**Terminal Output**:
```
 ✓ src/components/PaymentForm.test.tsx (5)
 ✓ src/components/PaymentConfirmation.test.tsx (3)

 Test Files  2 passed (2)
      Tests  8 passed (8)
```

**All tests pass. You ship to production.**

**Production failure**:

**Browser Console**:
```
Uncaught TypeError: Cannot read properties of undefined (reading 'transactionId')
    at PaymentConfirmation.tsx:12
    at PaymentPage.tsx:45
```

**What happened**: `PaymentForm` calls `onSubmit` with payment data, but `PaymentPage` expects the response to include a `transactionId`. The unit tests mocked both sides independently, so they never caught the mismatch.

### Diagnostic Analysis: The Integration Gap

**Unit tests verified**:
- ✅ PaymentForm calls onSubmit with correct data
- ✅ PaymentConfirmation displays confirmation message

**Unit tests didn't verify**:
- ❌ PaymentForm and PaymentPage communicate correctly
- ❌ PaymentPage and PaymentConfirmation pass correct props
- ❌ The complete payment flow works end-to-end

**The problem**: Components work in isolation but fail when integrated. Unit tests can't catch this because they mock all dependencies.

**The solution**: Integration tests that render multiple components together.

## Integration Tests: Testing Component Collaboration

Integration tests render multiple components together to verify they work as a system:

```tsx
// src/pages/PaymentPage.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { PaymentPage } from './PaymentPage';

describe('PaymentPage - Integration', () => {
  it('completes full payment flow', async () => {
    const user = userEvent.setup();
    
    // Mock the API call
    const mockProcessPayment = vi.fn().mockResolvedValue({
      success: true,
      transactionId: 'txn_123456',
      amount: 99.99,
    });

    render(<PaymentPage processPayment={mockProcessPayment} />);

    // User fills out payment form
    await user.type(screen.getByLabelText('Card Number'), '4532015112830366');
    await user.type(screen.getByLabelText('Expiry Date'), '12/25');
    await user.type(screen.getByLabelText('CVV'), '123');
    await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');

    // User submits payment
    await user.click(screen.getByRole('button', { name: /pay \$99\.99/i }));

    // Verify API was called with correct data
    expect(mockProcessPayment).toHaveBeenCalledWith({
      cardNumber: '4532015112830366',
      expiryDate: '12/25',
      cvv: '123',
      cardholderName: 'John Doe',
    });

    // Verify confirmation screen appears
    expect(await screen.findByText(/payment successful/i)).toBeInTheDocument();
    expect(screen.getByText(/transaction id: txn_123456/i)).toBeInTheDocument();
    expect(screen.getByText(/amount: \$99\.99/i)).toBeInTheDocument();
  });
});
```

**What this test verifies**:
1. PaymentForm renders and accepts input
2. PaymentForm calls the API with correct data
3. PaymentPage handles the API response
4. PaymentConfirmation receives and displays correct data
5. The complete flow works end-to-end

**Run the test**:

**Terminal Output**:
```
 ✓ src/pages/PaymentPage.test.tsx (1)
   ✓ PaymentPage - Integration (1)
     ✓ completes full payment flow

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  14:23:45
   Duration  567ms
```

## Reference Implementation: Complete Payment Page

Let's build the complete payment page that our integration test verifies:

```tsx
// src/pages/PaymentPage.tsx
import { useState } from 'react';
import { PaymentForm } from '../components/PaymentForm';
import { PaymentConfirmation } from '../components/PaymentConfirmation';

interface PaymentPageProps {
  processPayment: (data: PaymentData) => Promise<PaymentResponse>;
}

interface PaymentData {
  cardNumber: string;
  expiryDate: string;
  cvv: string;
  cardholderName: string;
}

interface PaymentResponse {
  success: boolean;
  transactionId: string;
  amount: number;
}

export function PaymentPage({ processPayment }: PaymentPageProps) {
  const [paymentResult, setPaymentResult] = useState<PaymentResponse | null>(null);

  const handlePayment = async (data: PaymentData) => {
    const result = await processPayment(data);
    setPaymentResult(result);
  };

  if (paymentResult) {
    return (
      <PaymentConfirmation
        transactionId={paymentResult.transactionId}
        amount={paymentResult.amount}
      />
    );
  }

  return <PaymentForm onSubmit={handlePayment} amount={99.99} />;
}
```

```tsx
// src/components/PaymentConfirmation.tsx
interface PaymentConfirmationProps {
  transactionId: string;
  amount: number;
}

export function PaymentConfirmation({ transactionId, amount }: PaymentConfirmationProps) {
  return (
    <div className="confirmation">
      <h2>Payment Successful</h2>
      <p>Your payment has been processed.</p>
      <dl>
        <dt>Transaction ID:</dt>
        <dd>{transactionId}</dd>
        <dt>Amount:</dt>
        <dd>${amount.toFixed(2)}</dd>
      </dl>
    </div>
  );
}
```

## Iteration 1: Testing Error Flow Integration

Let's test what happens when payment fails:

```tsx
it('handles payment failure', async () => {
  const user = userEvent.setup();
  
  const mockProcessPayment = vi.fn().mockRejectedValue(
    new Error('Payment declined: Insufficient funds')
  );

  render(<PaymentPage processPayment={mockProcessPayment} />);

  // Fill and submit form
  await user.type(screen.getByLabelText('Card Number'), '4532015112830366');
  await user.type(screen.getByLabelText('Expiry Date'), '12/25');
  await user.type(screen.getByLabelText('CVV'), '123');
  await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
  await user.click(screen.getByRole('button', { name: /pay/i }));

  // Verify error message appears
  const errorMessage = await screen.findByRole('alert');
  expect(errorMessage).toHaveTextContent('Payment declined: Insufficient funds');

  // Verify we're still on payment form (not confirmation)
  expect(screen.getByLabelText('Card Number')).toBeInTheDocument();
  expect(screen.queryByText(/payment successful/i)).not.toBeInTheDocument();
});
```

**Run the test**:

**Terminal Output**:
```
 ✓ src/pages/PaymentPage.test.tsx (2)
   ✓ PaymentPage - Integration (2)
     ✓ completes full payment flow
     ✓ handles payment failure

 Test Files  1 passed (1)
      Tests  2 passed (2)
   Start at  14:35:12
   Duration  623ms
```

## Iteration 2: Testing Loading States Across Components

Let's verify that loading states propagate correctly:

```tsx
it('shows loading state during payment processing', async () => {
  const user = userEvent.setup();
  
  // Create a promise we can control
  let resolvePayment: (value: PaymentResponse) => void;
  const paymentPromise = new Promise<PaymentResponse>((resolve) => {
    resolvePayment = resolve;
  });
  
  const mockProcessPayment = vi.fn().mockReturnValue(paymentPromise);

  render(<PaymentPage processPayment={mockProcessPayment} />);

  // Fill and submit form
  await user.type(screen.getByLabelText('Card Number'), '4532015112830366');
  await user.type(screen.getByLabelText('Expiry Date'), '12/25');
  await user.type(screen.getByLabelText('CVV'), '123');
  await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
  await user.click(screen.getByRole('button', { name: /pay/i }));

  // Verify loading state
  expect(await screen.findByRole('button', { name: /processing/i })).toBeInTheDocument();
  expect(screen.getByRole('button', { name: /processing/i })).toBeDisabled();

  // Resolve the payment
  resolvePayment!({
    success: true,
    transactionId: 'txn_123456',
    amount: 99.99,
  });

  // Verify confirmation appears
  expect(await screen.findByText(/payment successful/i)).toBeInTheDocument();
});
```

**What this test verifies**:
1. Loading state appears immediately after submission
2. Submit button is disabled during processing
3. Confirmation appears after processing completes
4. The entire state transition works correctly

## Testing with React Router

When components use routing, integration tests need to provide router context:

```tsx
// src/pages/CheckoutFlow.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import { MemoryRouter, Routes, Route } from 'react-router-dom';
import userEvent from '@testing-library/user-event';
import { CheckoutFlow } from './CheckoutFlow';

describe('CheckoutFlow - Integration', () => {
  it('navigates through checkout steps', async () => {
    const user = userEvent.setup();
    
    render(
      <MemoryRouter initialEntries={['/checkout']}>
        <Routes>
          <Route path="/checkout" element={<CheckoutFlow />} />
          <Route path="/checkout/payment" element={<PaymentPage />} />
          <Route path="/checkout/confirmation" element={<ConfirmationPage />} />
        </Routes>
      </MemoryRouter>
    );

    // Start at shipping address step
    expect(screen.getByText(/shipping address/i)).toBeInTheDocument();

    // Fill shipping form
    await user.type(screen.getByLabelText('Street Address'), '123 Main St');
    await user.type(screen.getByLabelText('City'), 'San Francisco');
    await user.type(screen.getByLabelText('ZIP Code'), '94102');
    await user.click(screen.getByRole('button', { name: /continue to payment/i }));

    // Verify navigation to payment step
    expect(await screen.findByText(/payment details/i)).toBeInTheDocument();

    // Fill payment form
    await user.type(screen.getByLabelText('Card Number'), '4532015112830366');
    await user.type(screen.getByLabelText('Expiry Date'), '12/25');
    await user.type(screen.getByLabelText('CVV'), '123');
    await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
    await user.click(screen.getByRole('button', { name: /complete purchase/i }));

    // Verify navigation to confirmation
    expect(await screen.findByText(/order confirmed/i)).toBeInTheDocument();
  });
});
```

**Key points**:
- Use `MemoryRouter` for testing (doesn't require browser)
- Set `initialEntries` to control starting route
- Render complete route structure
- Test navigation between routes

## Testing with Context Providers

When components use context, integration tests need to provide that context:

```tsx
// src/contexts/CartContext.test.tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { CartProvider } from './CartContext';
import { ProductList } from '../components/ProductList';
import { Cart } from '../components/Cart';

describe('Cart Integration', () => {
  it('adds products to cart and displays total', async () => {
    const user = userEvent.setup();
    
    render(
      <CartProvider>
        <ProductList />
        <Cart />
      </CartProvider>
    );

    // Initially cart is empty
    expect(screen.getByText(/cart is empty/i)).toBeInTheDocument();

    // Add first product
    const addButtons = screen.getAllByRole('button', { name: /add to cart/i });
    await user.click(addButtons[0]);

    // Verify product appears in cart
    expect(await screen.findByText(/product 1/i)).toBeInTheDocument();
    expect(screen.getByText(/\$29\.99/i)).toBeInTheDocument();

    // Add second product
    await user.click(addButtons[1]);

    // Verify total is calculated correctly
    expect(screen.getByText(/total: \$59\.98/i)).toBeInTheDocument();
  });
});
```

**What this test verifies**:
1. Multiple components share cart state via context
2. Adding products updates cart display
3. Cart total is calculated correctly
4. Context provider works as expected

## Testing with API Mocking

For integration tests that make API calls, use MSW (Mock Service Worker):

```bash
npm install -D msw
```

```typescript
// src/__tests__/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.post('/api/payment', async ({ request }) => {
    const body = await request.json();
    
    // Simulate payment processing
    await new Promise(resolve => setTimeout(resolve, 100));
    
    return HttpResponse.json({
      success: true,
      transactionId: 'txn_' + Math.random().toString(36).substr(2, 9),
      amount: 99.99,
    });
  }),

  http.get('/api/products', () => {
    return HttpResponse.json([
      { id: 1, name: 'Product 1', price: 29.99 },
      { id: 2, name: 'Product 2', price: 39.99 },
    ]);
  }),
];
```

```typescript
// src/__tests__/setup.ts
import { expect, afterEach, beforeAll, afterAll } from 'vitest';
import { cleanup } from '@testing-library/react';
import * as matchers from '@testing-library/jest-dom/matchers';
import { setupServer } from 'msw/node';
import { handlers } from './mocks/handlers';

expect.extend(matchers);

// Setup MSW server
const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterAll(() => server.close());
afterEach(() => {
  cleanup();
  server.resetHandlers();
});
```

Now integration tests can make real API calls that are intercepted by MSW:

```tsx
it('fetches and displays products', async () => {
  render(<ProductList />);

  // Wait for products to load
  expect(await screen.findByText('Product 1')).toBeInTheDocument();
  expect(screen.getByText('Product 2')).toBeInTheDocument();
  expect(screen.getByText('$29.99')).toBeInTheDocument();
  expect(screen.getByText('$39.99')).toBeInTheDocument();
});
```

**Benefits of MSW**:
- Tests use real fetch/axios calls (no mocking)
- Responses are realistic (proper HTTP structure)
- Can simulate network errors and delays
- Works in both tests and browser (for development)

## When to Write Integration Tests

**Write integration tests for**:
- ✅ Multi-step user flows (checkout, registration)
- ✅ Components that communicate via context
- ✅ Components that share state
- ✅ Navigation between routes
- ✅ API integration points

**Don't write integration tests for**:
- ❌ Simple components with no dependencies
- ❌ Pure functions (use unit tests)
- ❌ Third-party library behavior
- ❌ Every possible component combination

**The rule**: Write integration tests for critical user journeys. If a flow breaking would be a serious bug, write an integration test.

## Integration vs. Unit Tests: When to Use Each

**Unit Tests**:
- Fast (milliseconds)
- Test one component in isolation
- Mock all dependencies
- Specific failure messages
- Easy to write and maintain

**Integration Tests**:
- Slower (seconds)
- Test multiple components together
- Minimal mocking
- Catch integration bugs
- More realistic

**The balance**: 70% unit tests, 20% integration tests, 10% E2E tests.

**Example distribution for a payment feature**:
- **Unit tests (7)**: PaymentForm validation, formatCardNumber, validateCVV, etc.
- **Integration tests (2)**: Complete payment flow, error handling flow
- **E2E test (1)**: Full checkout in real browser

This gives you confidence without excessive test maintenance.

## Playwright for E2E (when necessary)

## The Failure: Integration Tests Miss Browser-Specific Bugs

Your integration tests pass. Your unit tests pass. You deploy to production.

**Production bug report**:
> "Payment form doesn't work in Safari. Submit button does nothing."

You investigate. The form works in Chrome. Works in Firefox. Fails in Safari.

### Diagnostic Analysis: The Browser Compatibility Gap

**What integration tests verified**:
- ✅ Form validation logic works
- ✅ API calls are made correctly
- ✅ State updates propagate
- ✅ Components render expected output

**What integration tests didn't verify**:
- ❌ Form works in actual Safari browser
- ❌ CSS doesn't break layout in Safari
- ❌ JavaScript features are supported in Safari
- ❌ Form submission works with real browser events

**The problem**: Integration tests run in jsdom, a simulated browser environment. jsdom doesn't perfectly replicate real browser behavior, especially browser-specific quirks.

**The solution**: End-to-end (E2E) tests that run in real browsers.

## When You Actually Need E2E Tests

**E2E tests are expensive**:
- Slow to run (minutes vs. milliseconds)
- Flaky (network issues, timing problems)
- Expensive to maintain (break on UI changes)
- Require infrastructure (browsers, servers)

**Write E2E tests only for**:
- ✅ Critical user flows (checkout, payment, registration)
- ✅ Browser-specific features (file uploads, camera access)
- ✅ Third-party integrations (OAuth, payment processors)
- ✅ Features that have failed in production before

**Don't write E2E tests for**:
- ❌ Every feature (too slow, too expensive)
- ❌ Edge cases (use integration tests)
- ❌ Unit-level logic (use unit tests)
- ❌ Styling and layout (use visual regression tools)

**The rule**: If integration tests give you 95% confidence, E2E tests give you the final 5%. Use them sparingly.

## Playwright: Modern E2E Testing

Playwright is a modern E2E testing framework that runs tests in real browsers (Chrome, Firefox, Safari, Edge).

### Installation

```bash
npm install -D @playwright/test
npx playwright install
```

This installs Playwright and downloads browser binaries.

### Configuration

Create `playwright.config.ts`:

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
});
```

**Configuration explained**:
- `testDir` - Where E2E tests live
- `projects` - Run tests in multiple browsers
- `webServer` - Automatically start dev server
- `trace` - Record test execution for debugging
- `screenshot` - Capture screenshots on failure

### Project Structure

```bash
e2e/
├── payment.spec.ts          ← E2E test for payment flow
├── checkout.spec.ts         ← E2E test for checkout flow
└── fixtures/
    └── test-data.ts         ← Shared test data
```

## Your First E2E Test: Payment Flow

Let's write an E2E test for the complete payment flow:

```typescript
// e2e/payment.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Payment Flow', () => {
  test('completes payment successfully', async ({ page }) => {
    // Navigate to payment page
    await page.goto('/checkout/payment');

    // Fill out payment form
    await page.getByLabel('Card Number').fill('4532015112830366');
    await page.getByLabel('Expiry Date').fill('12/25');
    await page.getByLabel('CVV').fill('123');
    await page.getByLabel('Cardholder Name').fill('John Doe');

    // Submit payment
    await page.getByRole('button', { name: /pay/i }).click();

    // Wait for confirmation page
    await expect(page.getByText(/payment successful/i)).toBeVisible();
    await expect(page.getByText(/transaction id:/i)).toBeVisible();

    // Verify URL changed
    await expect(page).toHaveURL(/\/checkout\/confirmation/);
  });
});
```

**Run the test**:

```bash
npx playwright test
```

**Terminal Output**:
```
Running 3 tests using 3 workers

  ✓  [chromium] › payment.spec.ts:3:3 › Payment Flow › completes payment successfully (2.3s)
  ✓  [firefox] › payment.spec.ts:3:3 › Payment Flow › completes payment successfully (2.1s)
  ✓  [webkit] › payment.spec.ts:3:3 › Payment Flow › completes payment successfully (2.5s)

  3 passed (7.2s)
```

**What happened**:
1. Playwright started your dev server
2. Opened Chrome, Firefox, and Safari
3. Ran the test in all three browsers
4. Verified the payment flow works in each

## Playwright vs. React Testing Library

**Key differences**:

| Aspect | React Testing Library | Playwright |
|--------|----------------------|------------|
| **Environment** | jsdom (simulated) | Real browsers |
| **Speed** | Fast (milliseconds) | Slow (seconds) |
| **Scope** | Component-level | Full application |
| **Network** | Mocked | Real or mocked |
| **Browser APIs** | Limited | Full support |
| **Flakiness** | Rare | More common |
| **Cost** | Low | High |

**Use React Testing Library for**: Component behavior, user interactions, state management

**Use Playwright for**: Critical flows, browser compatibility, third-party integrations

## Iteration 1: Testing Error Scenarios

Let's test what happens when payment fails:

```typescript
test('displays error when payment fails', async ({ page }) => {
  // Mock API to return error
  await page.route('**/api/payment', async (route) => {
    await route.fulfill({
      status: 400,
      contentType: 'application/json',
      body: JSON.stringify({
        error: 'Payment declined: Insufficient funds',
      }),
    });
  });

  await page.goto('/checkout/payment');

  // Fill and submit form
  await page.getByLabel('Card Number').fill('4532015112830366');
  await page.getByLabel('Expiry Date').fill('12/25');
  await page.getByLabel('CVV').fill('123');
  await page.getByLabel('Cardholder Name').fill('John Doe');
  await page.getByRole('button', { name: /pay/i }).click();

  // Verify error message appears
  await expect(page.getByRole('alert')).toContainText('Insufficient funds');

  // Verify we're still on payment page
  await expect(page).toHaveURL(/\/checkout\/payment/);
  
  // Verify form is still editable
  await expect(page.getByLabel('Card Number')).toBeEditable();
});
```

**What this test verifies**:
1. Error message displays in real browser
2. User stays on payment page
3. Form remains editable for retry
4. Error handling works across browsers

## Iteration 2: Testing Browser-Specific Features

Let's test file upload (a feature that requires real browser):

```typescript
test('uploads receipt image', async ({ page }) => {
  await page.goto('/checkout/payment');

  // Fill payment form
  await page.getByLabel('Card Number').fill('4532015112830366');
  await page.getByLabel('Expiry Date').fill('12/25');
  await page.getByLabel('CVV').fill('123');
  await page.getByLabel('Cardholder Name').fill('John Doe');

  // Upload receipt
  const fileInput = page.getByLabel('Upload Receipt (optional)');
  await fileInput.setInputFiles('./e2e/fixtures/receipt.jpg');

  // Verify preview appears
  await expect(page.getByAltText('Receipt preview')).toBeVisible();

  // Submit payment
  await page.getByRole('button', { name: /pay/i }).click();

  // Verify receipt was uploaded
  await expect(page.getByText(/receipt uploaded/i)).toBeVisible();
});
```

**Why this needs E2E**:
- File upload requires real browser file system
- Image preview requires real image rendering
- jsdom can't simulate this accurately

## Iteration 3: Testing Third-Party Integrations

Let's test Stripe payment integration:

```typescript
test('processes payment through Stripe', async ({ page }) => {
  await page.goto('/checkout/payment');

  // Fill payment form
  await page.getByLabel('Card Number').fill('4242424242424242'); // Stripe test card
  await page.getByLabel('Expiry Date').fill('12/25');
  await page.getByLabel('CVV').fill('123');
  await page.getByLabel('Cardholder Name').fill('Test User');

  // Submit payment
  await page.getByRole('button', { name: /pay/i }).click();

  // Wait for Stripe iframe to load
  const stripeFrame = page.frameLocator('iframe[name^="__privateStripeFrame"]');
  
  // Verify Stripe processed payment
  await expect(page.getByText(/payment successful/i)).toBeVisible({ timeout: 10000 });
  
  // Verify transaction ID from Stripe
  await expect(page.getByText(/pi_/)).toBeVisible(); // Stripe payment intent ID
});
```

**Why this needs E2E**:
- Stripe loads in iframe (can't test in jsdom)
- Real network calls to Stripe API
- Verifies actual integration works

## Debugging Failed E2E Tests

When E2E tests fail, Playwright provides powerful debugging tools:

### 1. Screenshots on Failure

Playwright automatically captures screenshots when tests fail:

```bash
npx playwright test
```

**Terminal Output**:
```
  ✗  [chromium] › payment.spec.ts:3:3 › Payment Flow › completes payment successfully (2.3s)

  Error: expect(locator).toBeVisible()

  Call log:
    - expect.toBeVisible with timeout 5000ms
    - waiting for getByText(/payment successful/i)

  Screenshot: test-results/payment-Payment-Flow-completes-payment-successfully-chromium/test-failed-1.png
```

The screenshot shows exactly what the browser looked like when the test failed.

### 2. Trace Viewer

Playwright records a trace of test execution:

```bash
npx playwright test --trace on
npx playwright show-report
```

This opens a UI showing:
- Every action taken
- Screenshots at each step
- Network requests
- Console logs
- DOM snapshots

### 3. Debug Mode

Run tests in debug mode to step through them:

```bash
npx playwright test --debug
```

This opens Playwright Inspector where you can:
- Step through test line by line
- Inspect page state at each step
- Try selectors in real-time
- See what the test sees

### 4. Headed Mode

Run tests with visible browser:

```bash
npx playwright test --headed
```

Watch the test run in a real browser window. Useful for understanding what's happening.

## Common E2E Test Patterns

### Waiting for Navigation

```typescript
// Wait for URL to change
await page.waitForURL('**/confirmation');

// Wait for navigation to complete
await page.waitForLoadState('networkidle');
```

### Handling Dialogs

```typescript
// Accept confirmation dialog
page.on('dialog', dialog => dialog.accept());
await page.getByRole('button', { name: /delete/i }).click();
```

### Testing Responsive Design

```typescript
test('works on mobile', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 667 });
  await page.goto('/checkout/payment');
  
  // Test mobile-specific behavior
  await expect(page.getByRole('button', { name: /menu/i })).toBeVisible();
});
```

### Testing Keyboard Navigation

```typescript
test('supports keyboard navigation', async ({ page }) => {
  await page.goto('/checkout/payment');
  
  // Tab through form fields
  await page.keyboard.press('Tab');
  await expect(page.getByLabel('Card Number')).toBeFocused();
  
  await page.keyboard.press('Tab');
  await expect(page.getByLabel('Expiry Date')).toBeFocused();
});
```

### Testing Accessibility

```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('has no accessibility violations', async ({ page }) => {
  await page.goto('/checkout/payment');
  
  const accessibilityScanResults = await new AxeBuilder({ page }).analyze();
  
  expect(accessibilityScanResults.violations).toEqual([]);
});
```

## E2E Test Maintenance: Keeping Tests Stable

E2E tests are notoriously flaky. Here's how to keep them stable:

### 1. Use Stable Selectors

**Bad** (brittle):

```typescript
// ❌ Breaks when CSS changes
await page.locator('.btn-primary').click();

// ❌ Breaks when text changes
await page.locator('text=Submit Payment').click();
```

**Good** (stable):

```typescript
// ✅ Uses semantic role
await page.getByRole('button', { name: /pay/i }).click();

// ✅ Uses label association
await page.getByLabel('Card Number').fill('4242424242424242');

// ✅ Uses test ID as last resort
await page.getByTestId('payment-submit').click();
```

### 2. Wait for Conditions, Not Timeouts

**Bad** (flaky):

```typescript
// ❌ Arbitrary timeout
await page.waitForTimeout(2000);
await page.getByText('Success').click();
```

**Good** (reliable):

```typescript
// ✅ Wait for specific condition
await page.getByText('Success').waitFor({ state: 'visible' });
await page.getByText('Success').click();

// ✅ Or use expect with auto-waiting
await expect(page.getByText('Success')).toBeVisible();
```

### 3. Isolate Tests

**Bad** (tests depend on each other):

```typescript
// ❌ Test 2 depends on Test 1
test('creates account', async ({ page }) => {
  // Creates user
});

test('logs in', async ({ page }) => {
  // Assumes user exists from previous test
});
```

**Good** (tests are independent):

```typescript
// ✅ Each test sets up its own data
test('logs in', async ({ page }) => {
  // Create user via API
  await createTestUser();
  
  // Now test login
  await page.goto('/login');
  // ...
});
```

### 4. Use Page Object Model

Encapsulate page interactions in reusable classes:

```typescript
// e2e/pages/PaymentPage.ts
export class PaymentPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/checkout/payment');
  }

  async fillCardNumber(cardNumber: string) {
    await this.page.getByLabel('Card Number').fill(cardNumber);
  }

  async fillExpiryDate(expiryDate: string) {
    await this.page.getByLabel('Expiry Date').fill(expiryDate);
  }

  async fillCVV(cvv: string) {
    await this.page.getByLabel('CVV').fill(cvv);
  }

  async fillCardholderName(name: string) {
    await this.page.getByLabel('Cardholder Name').fill(name);
  }

  async submit() {
    await this.page.getByRole('button', { name: /pay/i }).click();
  }

  async expectSuccess() {
    await expect(this.page.getByText(/payment successful/i)).toBeVisible();
  }
}
```

```typescript
// e2e/payment.spec.ts
import { PaymentPage } from './pages/PaymentPage';

test('completes payment', async ({ page }) => {
  const paymentPage = new PaymentPage(page);
  
  await paymentPage.goto();
  await paymentPage.fillCardNumber('4532015112830366');
  await paymentPage.fillExpiryDate('12/25');
  await paymentPage.fillCVV('123');
  await paymentPage.fillCardholderName('John Doe');
  await paymentPage.submit();
  await paymentPage.expectSuccess();
});
```

**Benefits**:
- Tests are more readable
- Page changes only require updating one place
- Reusable across multiple tests

## When to Apply: E2E Testing Decision Framework

**Write E2E tests for**:
- ✅ Critical user flows (checkout, payment, registration)
- ✅ Features that have failed in production
- ✅ Browser-specific functionality (file upload, camera)
- ✅ Third-party integrations (OAuth, Stripe)
- ✅ Flows that span multiple pages

**Don't write E2E tests for**:
- ❌ Every feature (too slow, too expensive)
- ❌ Unit-level logic (use unit tests)
- ❌ Component behavior (use integration tests)
- ❌ Edge cases (use integration tests)
- ❌ Styling (use visual regression tools)

**The rule**: E2E tests are your last line of defense. Use them for the 5% of functionality where integration tests aren't enough.

## The Complete Testing Strategy

For a production application, use all three types of tests:

**Example: Payment Feature**

**Unit Tests (70%)**:
- `validateCardNumber()` - 6 tests for edge cases
- `formatExpiryDate()` - 4 tests for formatting
- `calculateTotal()` - 5 tests for calculations
- `PaymentForm` component - 8 tests for behavior

**Integration Tests (20%)**:
- Complete payment flow - 1 test
- Error handling flow - 1 test
- Loading states - 1 test
- Cart integration - 1 test

**E2E Tests (10%)**:
- End-to-end checkout in Chrome - 1 test
- End-to-end checkout in Safari - 1 test

**Total**: 23 unit tests, 4 integration tests, 2 E2E tests

**Run time**:
- Unit tests: 2 seconds
- Integration tests: 5 seconds
- E2E tests: 15 seconds
- **Total**: 22 seconds

This gives you comprehensive coverage without excessive maintenance burden.

## The Complete Testing Journey

## The Journey: From No Tests to Comprehensive Coverage

Let's trace the evolution of our payment form testing strategy:

| Phase | Testing Approach | What We Caught | What We Missed | Run Time |
|-------|-----------------|----------------|----------------|----------|
| **0. No Tests** | Manual testing only | Nothing automatically | Everything | N/A |
| **1. Basic Unit Tests** | Component renders | Rendering crashes | User interactions, integration bugs | 0.5s |
| **2. Interaction Tests** | User can fill form | Form validation, submission | Integration with API, browser bugs | 2s |
| **3. Integration Tests** | Complete payment flow | API integration, state management | Browser-specific bugs | 7s |
| **4. E2E Tests** | Real browser testing | Safari compatibility, file upload | Nothing (comprehensive) | 22s |

## Final Implementation: Complete Test Suite

Here's the complete test suite for our payment form:

### Unit Tests (PaymentForm.test.tsx)

```tsx
// src/components/PaymentForm.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { PaymentForm } from './PaymentForm';

describe('PaymentForm', () => {
  describe('Rendering', () => {
    it('renders all form fields', () => {
      render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);
      
      expect(screen.getByLabelText('Card Number')).toBeInTheDocument();
      expect(screen.getByLabelText('Expiry Date')).toBeInTheDocument();
      expect(screen.getByLabelText('CVV')).toBeInTheDocument();
      expect(screen.getByLabelText('Cardholder Name')).toBeInTheDocument();
      expect(screen.getByRole('button', { name: /pay \$99\.99/i })).toBeInTheDocument();
    });

    it('displays correct amount', () => {
      render(<PaymentForm onSubmit={vi.fn()} amount={149.99} />);
      
      expect(screen.getByText('Amount: $149.99')).toBeInTheDocument();
      expect(screen.getByRole('button', { name: /pay \$149\.99/i })).toBeInTheDocument();
    });
  });

  describe('Validation', () => {
    it('disables submit button when form is empty', () => {
      render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);
      
      expect(screen.getByRole('button', { name: /pay/i })).toBeDisabled();
    });

    it('enables submit button when all fields are valid', async () => {
      const user = userEvent.setup();
      render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);
      
      await user.type(screen.getByLabelText('Card Number'), '4532015112830366');
      await user.type(screen.getByLabelText('Expiry Date'), '12/25');
      await user.type(screen.getByLabelText('CVV'), '123');
      await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
      
      expect(screen.getByRole('button', { name: /pay/i })).toBeEnabled();
    });

    it('keeps button disabled with invalid card number', async () => {
      const user = userEvent.setup();
      render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);
      
      await user.type(screen.getByLabelText('Card Number'), '12345'); // Too short
      await user.type(screen.getByLabelText('Expiry Date'), '12/25');
      await user.type(screen.getByLabelText('CVV'), '123');
      await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
      
      expect(screen.getByRole('button', { name: /pay/i })).toBeDisabled();
    });
  });

  describe('Submission', () => {
    it('calls onSubmit with form data', async () => {
      const user = userEvent.setup();
      const mockOnSubmit = vi.fn().mockResolvedValue(undefined);
      
      render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);
      
      await user.type(screen.getByLabelText('Card Number'), '4532015112830366');
      await user.type(screen.getByLabelText('Expiry Date'), '12/25');
      await user.type(screen.getByLabelText('CVV'), '123');
      await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
      await user.click(screen.getByRole('button', { name: /pay/i }));
      
      expect(mockOnSubmit).toHaveBeenCalledWith({
        cardNumber: '4532015112830366',
        expiryDate: '12/25',
        cvv: '123',
        cardholderName: 'John Doe',
      });
    });

    it('shows loading state during submission', async () => {
      const user = userEvent.setup();
      const mockOnSubmit = vi.fn().mockImplementation(
        () => new Promise(resolve => setTimeout(resolve, 100))
      );
      
      render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);
      
      await user.type(screen.getByLabelText('Card Number'), '4532015112830366');
      await user.type(screen.getByLabelText('Expiry Date'), '12/25');
      await user.type(screen.getByLabelText('CVV'), '123');
      await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
      await user.click(screen.getByRole('button', { name: /pay/i }));
      
      expect(await screen.findByRole('button', { name: /processing/i })).toBeDisabled();
    });

    it('displays error message on failure', async () => {
      const user = userEvent.setup();
      const mockOnSubmit = vi.fn().mockRejectedValue(
        new Error('Payment declined')
      );
      
      render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);
      
      await user.type(screen.getByLabelText('Card Number'), '4532015112830366');
      await user.type(screen.getByLabelText('Expiry Date'), '12/25');
      await user.type(screen.getByLabelText('CVV'), '123');
      await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
      await user.click(screen.getByRole('button', { name: /pay/i }));
      
      expect(await screen.findByRole('alert')).toHaveTextContent('Payment declined');
    });
  });

  describe('Accessibility', () => {
    it('has accessible form labels', () => {
      render(<PaymentForm onSubmit={vi.fn()} amount={99.99} />);
      
      expect(screen.getByLabelText('Card Number')).toBeInTheDocument();
      expect(screen.getByLabelText('Expiry Date')).toBeInTheDocument();
      expect(screen.getByLabelText('CVV')).toBeInTheDocument();
      expect(screen.getByLabelText('Cardholder Name')).toBeInTheDocument();
    });

    it('announces errors to screen readers', async () => {
      const user = userEvent.setup();
      const mockOnSubmit = vi.fn().mockRejectedValue(new Error('Error'));
      
      render(<PaymentForm onSubmit={mockOnSubmit} amount={99.99} />);
      
      await user.type(screen.getByLabelText('Card Number'), '4532015112830366');
      await user.type(screen.getByLabelText('Expiry Date'), '12/25');
      await user.type(screen.getByLabelText('CVV'), '123');
      await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
      await user.click(screen.getByRole('button', { name: /pay/i }));
      
      const error = await screen.findByRole('alert');
      expect(error).toBeInTheDocument();
    });
  });
});
```

### Unit Tests (validation.test.ts)

```typescript
// src/lib/validation.test.ts
import { describe, it, expect } from 'vitest';
import { validateCardNumber, validateExpiryDate, validateCVV } from './validation';

describe('validateCardNumber', () => {
  it('accepts valid card numbers', () => {
    expect(validateCardNumber('4532015112830366')).toBe(true);
    expect(validateCardNumber('6011514433546201')).toBe(true);
  });

  it('rejects invalid card numbers', () => {
    expect(validateCardNumber('1234567812345678')).toBe(false);
  });

  it('handles spaces and dashes', () => {
    expect(validateCardNumber('4532 0151 1283 0366')).toBe(true);
    expect(validateCardNumber('4532-0151-1283-0366')).toBe(true);
  });

  it('rejects wrong length', () => {
    expect(validateCardNumber('453201511283036')).toBe(false);
    expect(validateCardNumber('45320151128303666')).toBe(false);
  });
});

describe('validateExpiryDate', () => {
  it('accepts valid future dates', () => {
    expect(validateExpiryDate('12/25')).toBe(true);
    expect(validateExpiryDate('01/30')).toBe(true);
  });

  it('rejects past dates', () => {
    expect(validateExpiryDate('01/20')).toBe(false);
  });

  it('rejects invalid format', () => {
    expect(validateExpiryDate('13/25')).toBe(false); // Invalid month
    expect(validateExpiryDate('12/2025')).toBe(false); // Wrong format
  });
});

describe('validateCVV', () => {
  it('accepts 3-digit CVV', () => {
    expect(validateCVV('123')).toBe(true);
  });

  it('accepts 4-digit CVV (Amex)', () => {
    expect(validateCVV('1234')).toBe(true);
  });

  it('rejects invalid CVV', () => {
    expect(validateCVV('12')).toBe(false);
    expect(validateCVV('12345')).toBe(false);
    expect(validateCVV('abc')).toBe(false);
  });
});
```

### Integration Tests (PaymentPage.test.tsx)

```tsx
// src/pages/PaymentPage.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { PaymentPage } from './PaymentPage';

describe('PaymentPage - Integration', () => {
  it('completes full payment flow', async () => {
    const user = userEvent.setup();
    const mockProcessPayment = vi.fn().mockResolvedValue({
      success: true,
      transactionId: 'txn_123456',
      amount: 99.99,
    });

    render(<PaymentPage processPayment={mockProcessPayment} />);

    // Fill form
    await user.type(screen.getByLabelText('Card Number'), '4532015112830366');
    await user.type(screen.getByLabelText('Expiry Date'), '12/25');
    await user.type(screen.getByLabelText('CVV'), '123');
    await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');

    // Submit
    await user.click(screen.getByRole('button', { name: /pay/i }));

    // Verify API call
    expect(mockProcessPayment).toHaveBeenCalledWith({
      cardNumber: '4532015112830366',
      expiryDate: '12/25',
      cvv: '123',
      cardholderName: 'John Doe',
    });

    // Verify confirmation
    expect(await screen.findByText(/payment successful/i)).toBeInTheDocument();
    expect(screen.getByText(/txn_123456/i)).toBeInTheDocument();
  });

  it('handles payment failure', async () => {
    const user = userEvent.setup();
    const mockProcessPayment = vi.fn().mockRejectedValue(
      new Error('Insufficient funds')
    );

    render(<PaymentPage processPayment={mockProcessPayment} />);

    await user.type(screen.getByLabelText('Card Number'), '4532015112830366');
    await user.type(screen.getByLabelText('Expiry Date'), '12/25');
    await user.type(screen.getByLabelText('CVV'), '123');
    await user.type(screen.getByLabelText('Cardholder Name'), 'John Doe');
    await user.click(screen.getByRole('button', { name: /pay/i }));

    expect(await screen.findByRole('alert')).toHaveTextContent('Insufficient funds');
    expect(screen.queryByText(/payment successful/i)).not.toBeInTheDocument();
  });
});
```

### E2E Tests (payment.spec.ts)

```typescript
// e2e/payment.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Payment Flow - E2E', () => {
  test('completes payment in Chrome', async ({ page }) => {
    await page.goto('/checkout/payment');

    await page.getByLabel('Card Number').fill('4532015112830366');
    await page.getByLabel('Expiry Date').fill('12/25');
    await page.getByLabel('CVV').fill('123');
    await page.getByLabel('Cardholder Name').fill('John Doe');
    await page.getByRole('button', { name: /pay/i }).click();

    await expect(page.getByText(/payment successful/i)).toBeVisible();
    await expect(page).toHaveURL(/\/checkout\/confirmation/);
  });

  test('handles payment failure', async ({ page }) => {
    await page.route('**/api/payment', async (route) => {
      await route.fulfill({
        status: 400,
        body: JSON.stringify({ error: 'Payment declined' }),
      });
    });

    await page.goto('/checkout/payment');

    await page.getByLabel('Card Number').fill('4532015112830366');
    await page.getByLabel('Expiry Date').fill('12/25');
    await page.getByLabel('CVV').fill('123');
    await page.getByLabel('Cardholder Name').fill('John Doe');
    await page.getByRole('button', { name: /pay/i }).click();

    await expect(page.getByRole('alert')).toContainText('Payment declined');
  });
});
```

## Decision Framework: Which Test Type When?

Use this flowchart to decide which type of test to write:

```
Is it a pure function with no dependencies?
├─ Yes → Unit test (Vitest)
└─ No → Continue

Does it involve multiple components working together?
├─ Yes → Integration test (React Testing Library)
└─ No → Continue

Does it require real browser behavior?
├─ Yes → E2E test (Playwright)
└─ No → Unit test (React Testing Library)
```

**Specific scenarios**:

| Scenario | Test Type | Tool |
|----------|-----------|------|
| Validation function | Unit | Vitest |
| Single component behavior | Unit | React Testing Library |
| Form submission flow | Integration | React Testing Library |
| Multi-page checkout | E2E | Playwright |
| File upload | E2E | Playwright |
| API integration | Integration | React Testing Library + MSW |
| Browser compatibility | E2E | Playwright |
| Accessibility | Unit/Integration | React Testing Library |

## Lessons Learned: From No Tests to Comprehensive Coverage

### 1. Start with High-Value Tests

Don't try to test everything at once. Start with:
- Critical user flows (payment, registration)
- Complex business logic (calculations, validations)
- Features that have broken before

### 2. Test Behavior, Not Implementation

Tests should verify what users experience, not how code works internally. If you can refactor without changing behavior, tests shouldn't break.

### 3. Use the Right Tool for the Job

- **Unit tests** for logic and isolated components
- **Integration tests** for component collaboration
- **E2E tests** for critical flows in real browsers

### 4. Make Tests Readable

Tests are documentation. A developer should be able to read a test and understand what the feature does.

### 5. Keep Tests Fast

Slow tests don't get run. Optimize for speed:
- Mock external dependencies
- Use parallel execution
- Run E2E tests only in CI

### 6. Maintain Tests Like Production Code

Tests are code. They need:
- Clear naming
- DRY principles (Page Object Model)
- Refactoring when they become brittle
- Regular maintenance

### 7. Coverage Is a Guide, Not a Goal

Don't aim for 100% coverage. Aim for:
- High coverage of critical paths
- Confidence in deployments
- Fast feedback on regressions

## The Professional React Developer's Testing Mindset

**Before writing code**:
- "How will I test this?"
- "What could go wrong?"
- "What's the user experience?"

**While writing code**:
- Write the test first (TDD) or immediately after
- Run tests frequently
- Fix failing tests before moving on

**Before deploying**:
- All tests pass
- Coverage meets team standards
- Critical flows have E2E tests

**After deployment**:
- Monitor for failures
- Add tests for production bugs
- Refactor tests as code evolves

**The goal**: Ship with confidence. Tests are your safety net.
