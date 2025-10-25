# Chapter 22: Advanced TypeScript Patterns in React

## Discriminated Unions for Props

## Learning Objective

Use discriminated unions to create components that accept different, mutually exclusive sets of props based on the value of a single "discriminant" prop.

## Why This Matters

Components often have multiple "modes" or "variants." A naive approach might use a series of optional props, but this can lead to impossible or invalid combinations. A discriminated union creates a type-safe contract that ensures only valid prop combinations are possible, catching bugs at compile time and making the component's API much clearer.

## Discovery Phase

Let's build a `StatusDisplay` component. It needs to show one of three states: loading, success, or error. Each state requires different data.

A naive approach might look like this:

```jsx
// The flawed, non-discriminated approach
type StatusDisplayProps_Flawed = {
  status: 'loading' | 'success' | 'error';
  data?: string; // Only for success
  errorMessage?: string; // Only for error
};

// What if someone passes `data` when status is 'error'?
// <StatusDisplay status="error" data="some data" /> // This is allowed, but nonsensical.
```

This is risky because nothing prevents a developer from passing contradictory props. A discriminated union solves this by tying the other props' existence to the value of the `status` prop.

```jsx
import React from 'react';

// Step 1: Define the individual types for each state.
// The `status` property is the "discriminant".
type LoadingState = { status: 'loading' };
type SuccessState = { status: 'success'; data: string };
type ErrorState = { status: 'error'; errorMessage: string };

// Step 2: Create a union of the state types.
type StatusDisplayProps = LoadingState | SuccessState | ErrorState;

// Step 3: Use the union type in the component.
function StatusDisplay(props: StatusDisplayProps) {
  // TypeScript's control flow analysis (narrowing) is key here.
  switch (props.status) {
    case 'loading':
      return <div>Loading...</div>;
    case 'success':
      // Inside this block, TS knows `props` is of type `SuccessState`.
      // Accessing `props.data` is safe.
      // Accessing `props.errorMessage` would be a compile-time error.
      return <div>Success! Data: {props.data}</div>;
    case 'error':
      // Inside this block, TS knows `props` is of type `ErrorState`.
      // Accessing `props.errorMessage` is safe.
      return <div>Error: {props.errorMessage}</div>;
    default:
      return null;
  }
}

export default function App() {
  return (
    <div>
      <StatusDisplay status="loading" />
      <StatusDisplay status="success" data="User data loaded!" />
      <StatusDisplay status="error" errorMessage="Failed to fetch." />
      {/* <StatusDisplay status="success" errorMessage="This is an error!" /> */}
      {/* The line above causes a TypeScript error because `errorMessage`
          is not a valid prop when `status` is 'success'. */}
    </div>
  );
}
```

**Rendered Output**:

```
Loading...
Success! Data: User data loaded!
Error: Failed to fetch.
```

By checking the `status` prop, TypeScript is smart enough to "narrow" the type of `props` within each `case` block. This makes it impossible to access a prop that doesn't belong to the current state, providing perfect type safety.

## Deep Dive

### The Three Elements of a Discriminated Union

For this pattern to work, you need three things:

1.  **A Discriminant Property**: A property that is present on every type in the union, which has a literal type (e.g., `'loading'`, `'success'`). `status` is our discriminant.
2.  **A Union of Types**: An outer type that is a union of all the possible state shapes (`StatusDisplayProps`).
3.  **Type Narrowing**: Code that checks the discriminant property (like a `switch` statement or `if/else` chain), which allows TypeScript to infer the specific type within that block.

### Example: Polymorphic Button

This pattern is extremely powerful for creating components with distinct behaviors. Consider a `Button` that can either be a standard button with an `onClick` handler, or a link with an `href`. It should never be both.

```jsx
type ButtonAsButton = {
  as: 'button';
  onClick: () => void;
  // `href` is not allowed here
};

type ButtonAsLink = {
  as: 'a';
  href: string;
  // `onClick` is not allowed here
};

type ButtonProps = ButtonAsButton | ButtonAsLink;

function Button(props: ButtonProps) {
  if (props.as === 'a') {
    // TS knows `props` is `ButtonAsLink`, so `props.href` is safe.
    return <a href={props.href}>...</a>;
  }

  // TS knows `props` must be `ButtonAsButton`.
  return <button onClick={props.onClick}>...</button>;
}
```

Here, the `as` prop is the discriminant, ensuring that you provide either `href` or `onClick`, but never a mix of the two.

### Common Confusion: "Why not just check for the prop's existence?"

**You might think**: "Instead of a `status` prop, I could just check `if (props.data)` to know it's a success state."

**Actually**: This is less reliable and less explicit. What if the success data is `0` or an empty string `''`? These are "falsy" values, and `if (props.data)` would fail, leading to bugs. The discriminant provides a clear, unambiguous signal of the component's intended state.

**Why the confusion happens**: Checking for truthiness is a common JavaScript idiom.

**How to remember**: A discriminated union creates a _contract_. The discriminant prop is the explicit term of that contract that tells the component which mode it's in. Relying on the presence of other data props is implicit and brittle.

### Production Perspective

- **API Clarity**: This pattern makes your component's API self-documenting. It's immediately clear to consumers that the component has different, mutually exclusive modes of operation.
- **State Machine Modeling**: Discriminated unions are the perfect way to model state machines in TypeScript. The `status` prop represents the current state, and the union defines all possible states and their associated data.
- **Reducing Complexity**: By preventing impossible prop combinations, you reduce the number of edge cases you need to handle inside your component, making the implementation simpler and more robust.

## Polymorphic Components

## Learning Objective

Understand the concept of polymorphic components—components that can render as different underlying HTML elements or other React components.

## Why This Matters

In a mature design system, you want foundational components like `Box`, `Text`, or `Button` to be flexible. A `Text` component shouldn't be hardcoded to always render a `<p>` tag. You might need it to be an `<h1>`, a `<span>`, or a `<strong>`. Polymorphism allows you to build these flexible components while preserving the type safety of the underlying element's props.

## Discovery Phase

Let's start with a basic, non-polymorphic `Text` component.

```jsx
import React from 'react';

type TextProps = {
  children: React.ReactNode;
  color?: 'primary' | 'secondary';
};

// This component is not polymorphic. It ALWAYS renders a `<p>` tag.
function Text({ children, color }: TextProps) {
  const style = {
    color: color === 'primary' ? 'blue' : 'black',
  };
  return <p style={style}>{children}</p>;
}

// What if we want this to be a heading? We can't.
// What if we want to pass a `className`? We'd have to add it to `TextProps`.
```

This is rigid. To make it polymorphic, we need to allow the consumer to specify what element it should render as. This introduces two challenges:

1.  How do we tell the component which element to render? (The "as prop" pattern, which we'll cover next).
2.  How do we make TypeScript understand that if it renders as an `<a>` tag, it should accept an `href` prop, and if it renders as a `<button>`, it should accept a `disabled` prop?

The answer to the second question lies in generics and a special type from React: `React.ElementType`.

```javascript
import React from 'react';

// `React.ElementType` represents any valid thing that can be a component's type:
// - A string like 'div', 'p', 'a'
// - A React component function or class

// Let's define the props for a generic, polymorphic component.
// We introduce a generic type `C` that must be a valid element type.
type PolymorphicProps<C extends React.ElementType> = {
  as?: C; // The component to render as.
  children: React.ReactNode;
  // We'll add more props in the next section...
};

// The component function itself is also generic.
function PolymorphicComponent<C extends React.ElementType = 'span'>({
  as,
  children,
  ...restProps
}: PolymorphicProps<C>) {
  // If an `as` prop is provided, we use it. Otherwise, we default to 'span'.
  const Component = as || 'span';

  // We can render the `Component` and spread the rest of the props onto it.
  return <Component {...restProps}>{children}</Component>;
}
```

This is the conceptual foundation. We've created a component that can accept an `as` prop and will render that element. The generic `C extends React.ElementType` is the key piece of TypeScript magic that makes this possible.

However, we're still missing a crucial part: how do we type `...restProps` so that it correctly represents the props of the `Component` we're rendering? That's what we'll solve in the next section.

## Deep Dive

### The Building Blocks of Polymorphism

To create a fully-typed polymorphic component, we need to combine our own custom props with the props of the element being rendered. The full type is a bit complex, so let's build it up.

1.  **Our Component's Props**: These are the props specific to our component, like `color` in our `Text` example.
2.  **The `as` Prop**: The prop that determines the rendered element, typed as `C extends React.ElementType`.
3.  **The Underlying Element's Props**: We need to get all the valid props for the element `C`. React and TypeScript give us a utility for this: `React.ComponentPropsWithoutRef<C>`. This utility type takes a component or tag name `C` and returns a type representing all of its props, excluding the `ref` prop (which is handled specially).

The final props type will be an intersection of our props and the underlying element's props, while making sure we don't have collisions.

### Common Confusion: "What's the difference between `React.ElementType` and `React.ReactNode`?"

**You might think**: "They both seem to be about 'React things'."

**Actually**: They represent very different concepts.

- `React.ReactNode`: Represents something that can be **rendered**. It's the _output_ of a component. It's what goes inside JSX, like `<p>Hello</p>` or a string. This is what you use for the `children` prop.
- `React.ElementType`: Represents the **type** of a component. It's the _thing you render_. It's what goes in the tag position, like `p` in `<p />` or `MyComponent` in `<MyComponent />`. This is what you use for the `as` prop.

**Why the confusion happens**: The names are abstract and sound similar.

**How to remember**: `Node` is a piece of the rendered output tree. `Type` is the type of element you are creating.

### Production Perspective

- **Design System Primitives**: Polymorphism is the cornerstone of modern design system libraries (like Chakra UI, Radix, Material UI). It's used to create highly reusable primitive components like `Box`, `Flex`, `Grid`, and `Text` that provide a consistent API for layout and styling, regardless of the underlying semantic element.
- **Avoiding Prop Bloat**: Without polymorphism, you'd have to add every possible HTML attribute (`className`, `id`, `aria-label`, etc.) to every component's props. Polymorphism automatically makes all the standard attributes of the rendered element available in a type-safe way.

## As Prop Pattern

## Learning Objective

Implement the "as prop" pattern to build fully-typed, reusable polymorphic components.

## Why This Matters

The "as prop" is the standard, community-accepted convention for implementing polymorphic components. Mastering this pattern allows you to build flexible, professional-grade components that are a joy to use, as they provide full type safety and autocompletion for the underlying element's props.

## Discovery Phase

Let's build a fully-typed `Heading` component that can render as any heading level (`h1` through `h6`) using the `as` prop.

```jsx
import React from 'react';

// Step 1: Define the component's own specific props.
type HeadingOwnProps<C extends React.ElementType> = {
  as?: C;
  children: React.ReactNode;
  color?: 'primary' | 'secondary';
};

// Step 2: Create the final, combined props type.
// This is the core of the pattern.
type HeadingProps<C extends React.ElementType> = HeadingOwnProps<C> &
  // Omit our own props from the underlying element's props to avoid collisions.
  Omit<React.ComponentPropsWithoutRef<C>, keyof HeadingOwnProps<C>>;

// Step 3: Create the generic component.
// We can set a default element type, e.g., 'h2'.
function Heading<C extends React.ElementType = 'h2'>({
  as,
  children,
  color,
  ...restProps
}: HeadingProps<C>) {
  const Component = as || 'h2';

  const style = {
    color: color === 'primary' ? 'purple' : 'inherit',
  };

  // `restProps` is now correctly typed with the props of `Component`.
  return (
    <Component style={style} {...restProps}>
      {children}
    </Component>
  );
}

// --- USAGE ---
export default function App() {
  return (
    <div>
      {/* Renders an <h2> by default */}
      <Heading color="primary">Default Heading (h2)</Heading>

      {/* Renders an <h1> */}
      <Heading as="h1">Main Heading (h1)</Heading>

      {/* Renders as an anchor tag `<a>` and accepts `href`! */}
      <Heading as="a" href="https://react.dev">
        Link Heading (a)
      </Heading>
    </div>
  );
}
```

**Rendered Output**:

```html
<h2 style="color: purple;">Default Heading (h2)</h2>
<h1 style="color: inherit;">Main Heading (h1)</h1>
<a href="https://react.dev" style="color: inherit;">Link Heading (a)</a>
```

When you use this component in a TypeScript environment, you'll see that when you type `<Heading as="a" ...`, your editor will autocomplete the `href` prop. If you use `<Heading as="h1" ...`, it won't. This is the pattern in action!

## Deep Dive

### Deconstructing the `HeadingProps` Type

The props type is the most complex part. Let's break it down.
`type HeadingProps<C extends React.ElementType> = HeadingOwnProps<C> & Omit<React.ComponentPropsWithoutRef<C>, keyof HeadingOwnProps<C>>`

- **`HeadingOwnProps<C>`**: These are our custom props (`as`, `children`, `color`). We make this generic so it can include the `as?: C` prop.
- **`&`**: This is an intersection. We are combining our own props with the props of the underlying element.
- **`React.ComponentPropsWithoutRef<C>`**: This gets all the props for the element `C` (e.g., all props for `HTMLAnchorElement` if `C` is `'a'`).
- **`Omit<..., keyof HeadingOwnProps<C>>`**: This is a crucial step. It removes any of our custom prop names from the underlying element's props. This prevents conflicts. For example, if we had a prop named `style` and the underlying element also has a `style` prop, this `Omit` ensures our version takes precedence.

### The Component Signature

`function Heading<C extends React.ElementType = 'h2'>({ ... }: HeadingProps<C>)`

- **`<C extends React.ElementType = 'h2'>`**: This makes the component function itself generic. We also provide a default type for `C`. If the consumer doesn't provide an `as` prop, `C` will be `'h2'`, and the component will render an `h2` by default.

### Production Perspective

- **Design System Foundation**: This pattern is the workhorse of modern component libraries. Use it for any component that acts as a foundational building block for your UI, especially layout and typography primitives.
- **Accessibility**: Polymorphism is a powerful accessibility tool. It allows you to build a component with consistent styling (e.g., a `Button`) but render it as the most semantically appropriate element for the context (e.g., an `<a>` if it navigates, or a `<button>` if it performs an action).
- **Complexity Trade-off**: This pattern is powerful but adds complexity to your type definitions. For a simple, one-off component that will only ever be a `div`, it's overkill. Reserve it for components that are intended to be highly reusable and flexible.

## Conditional Types in Components

## Learning Objective

Use conditional types (`T extends U ? X : Y`) to create component props where the type of one prop depends on the value of another.

## Why This Matters

Sometimes, a component's behavior changes so dramatically based on a prop that other props need to change their types as well. Conditional types allow you to model these complex relationships, creating "smart" components that guide the developer toward correct usage and prevent logical errors.

## Discovery Phase

Let's build an `Input` component. If we pass `type="checkbox"`, the `value` should be a `boolean`. But if we pass `type="text"`, the `value` should be a `string`. A simple props definition can't handle this.

```jsx
// Flawed approach - `value` is a loose union type
type InputProps_Flawed = {
  type: 'text' | 'checkbox';
  value: string | boolean; // How do we know which it is?
  onChange: (value: string | boolean) => void;
};

// This allows invalid combinations:
// <Input type="text" value={true} ... />
```

This is unsafe. We can use a conditional type to link the `value` type to the `type` prop's value.

```jsx
import React from 'react';

// Step 1: Use a generic `T` for the `type` prop.
type InputProps<T extends 'text' | 'checkbox'> = {
  type: T;
  // Step 2: Define `value` using a conditional type.
  value: T extends 'checkbox' ? boolean : string;
  // Step 3: Do the same for the `onChange` callback.
  onChange: (value: T extends 'checkbox' ? boolean : string) => void;
};

// The component itself is generic.
function Input<T extends 'text' | 'checkbox'>(props: InputProps<T>) {
  const { type, value, onChange } = props;

  if (type === 'checkbox') {
    // Because of the conditional type, TS knows `value` is a boolean here
    // and `onChange` expects a boolean.
    return (
      <input
        type="checkbox"
        checked={value as boolean}
        onChange={(e) => onChange(e.target.checked as InputProps<T>['value'])}
      />
    );
  }

  // Here, TS knows `value` is a string and `onChange` expects a string.
  return (
    <input
      type="text"
      value={value as string}
      onChange={(e) => onChange(e.target.value as InputProps<T>['value'])}
    />
  );
}

// --- USAGE ---
export default function App() {
  const [text, setText] = React.useState('hello');
  const [checked, setChecked] = React.useState(true);

  return (
    <div>
      <Input type="text" value={text} onChange={setText} />
      {/* `onChange` correctly expects a string. Passing `setChecked` would be an error. */}

      <Input type="checkbox" value={checked} onChange={setChecked} />
      {/* `onChange` correctly expects a boolean. Passing `setText` would be an error. */}
    </div>
  );
}
```

**Interactive Behavior**:
A text input pre-filled with "hello" and a checked checkbox are displayed. They are both fully functional and type-safe.

The core of this pattern is the conditional type:
`T extends 'checkbox' ? boolean : string`

This reads as: "If the generic type `T` is `'checkbox'`, then this type is `boolean`. Otherwise, it is `string`." By making our `InputProps` and `Input` component generic over `T`, we link the types of `value` and `onChange` directly to the literal value passed to the `type` prop.

## Deep Dive

### Combining with Discriminated Unions

For more complex cases, you can combine conditional types with discriminated unions for maximum clarity and safety. This often leads to more readable code than a complex generic type.

```javascript
type TextInputProps = {
  type: 'text';
  value: string;
  onChange: (value: string) => void;
};

type CheckboxInputProps = {
  type: 'checkbox';
  value: boolean;
  onChange: (value: boolean) => void;
};

type BetterInputProps = TextInputProps | CheckboxInputProps;

function BetterInput(props: BetterInputProps) {
  if (props.type === 'checkbox') {
    // TS narrows `props` to `CheckboxInputProps`.
    // `props.value` is a boolean.
    return <input type="checkbox" checked={props.value} onChange={e => props.onChange(e.target.checked)} />;
  }

  // TS narrows `props` to `TextInputProps`.
  // `props.value` is a string.
  return <input type="text" value={props.value} onChange={e => props.onChange(e.target.value)} />;
}
```

This discriminated union approach achieves the same type safety as the conditional type example but can be easier to read and extend. The choice between them is often a matter of style and complexity. Conditional types shine when the relationship is more complex or involves many properties.

### Production Perspective

- **Highly Dynamic Components**: Use conditional types for components where the fundamental data type it operates on changes based on a prop. This is common in form libraries, data visualization components, or any component that wraps native HTML elements with varied behaviors.
- **Improving Developer Experience (DX)**: The main benefit is for the consumer of your component. They get immediate feedback if they provide an invalid combination of props. For example, ` <Input type="text" value={true} ... />` would be an instant error.
- **Readability vs. Power**: Conditional types are an advanced feature. While powerful, they can make your type definitions harder to understand at a glance. Always consider if a simpler pattern, like a discriminated union, can achieve the same goal with more readable code.

## Template Literal Types

## Learning Objective

Use template literal types to create precise and expressive string literal types, often for styling and layout props in a design system.

## Why This Matters

In a design system, you want to enforce consistency. Instead of allowing any string for a `margin` prop, you can use template literal types to restrict it to a combination of a direction, a size from your theme, and a unit. This provides powerful autocompletion and prevents developers from using "magic numbers" or inconsistent values.

## Discovery Phase

Let's build a `Box` component that should accept spacing props based on a predefined theme. We want props like `padding="md"` or `margin-top="lg"`.

```javascript
// Our design system's theme
const theme = {
  spacing: {
    sm: '0.5rem',
    md: '1rem',
    lg: '2rem',
  },
};

// Define types based on the theme
type Space = keyof typeof theme.spacing; // 'sm' | 'md' | 'lg'

// Now, let's define our prop types using template literals
type MarginProperty = 'margin' | 'margin-top' | 'margin-bottom' | 'margin-left' | 'margin-right';
type PaddingProperty = 'padding' | 'padding-top' | 'padding-bottom' | 'padding-left' | 'padding-right';

// Example of a template literal type
// This creates a union of strings like "margin-sm", "margin-top-lg", etc.
// This is not what we want for props, but shows the concept.
type SpacingClass = `${MarginProperty | PaddingProperty}-${Space}`;

// A better way for props is to use mapped types, which we'll see next.
// For now, let's create a simpler version for a single prop.
type BoxProps = {
    children: React.ReactNode;
    // A prop that accepts a size from our theme
    padding?: Space;
}
```

This is a good start, but template literals truly shine when combined with other types to generate a large set of possibilities. Let's define types for CSS properties that can take our themed spacing.

```jsx
import React from 'react';

const theme = {
  spacing: {
    sm: '0.5rem',
    md: '1rem',
    lg: '2rem',
  },
};

type Space = keyof typeof theme.spacing; // 'sm' | 'md' | 'lg'
type Direction = 'top' | 'bottom' | 'left' | 'right';

// Template literal type to generate margin property names
// Creates: 'marginTop' | 'marginBottom' | 'marginLeft' | 'marginRight'
type MarginDirectional = `margin${Capitalize<Direction>}`;

// Use a Mapped Type (see next section) to create the props
type BoxProps = {
  children: React.ReactNode;
} & {
  // For each property like 'marginTop', its value can be one of our space keys
  [Property in MarginDirectional]?: Space;
};

function Box({ children, ...styleProps }: BoxProps) {
  const style: React.CSSProperties = {};

  // A simple loop to transform our props into CSS
  for (const [key, value] of Object.entries(styleProps)) {
    if (key in styleMappings && value) {
      const cssProperty = styleMappings[key as keyof typeof styleMappings];
      style[cssProperty] = theme.spacing[value as Space];
    }
  }

  return <div style={style}>{children}</div>;
}

// Helper to map prop names to CSS property names
const styleMappings = {
  marginTop: 'marginTop',
  marginBottom: 'marginBottom',
  marginLeft: 'marginLeft',
  marginRight: 'marginRight',
};

// --- USAGE ---
export default function App() {
  return (
    <Box marginTop="lg" marginLeft="sm">
      I am a box with themed spacing!
    </Box>
  );
}
```

**Rendered Output**:

```html
<div style="margin-top: 2rem; margin-left: 0.5rem;">
  I am a box with themed spacing!
</div>
```

Here, we used ` `margin${Capitalize<Direction>}` ` to programmatically generate the type `'marginTop' | 'marginBottom' | ...`. This creates a type-safe API for our `Box` component. A developer using it gets autocompletion for `marginTop`, and if they provide a value, they get autocompletion for `'sm'`, `'md'`, and `'lg'`.

## Deep Dive

### Combining Unions in Template Literals

Template literals can expand unions of unions, creating a Cartesian product of all possible strings.

```typescript
type Size = "small" | "large";
type Color = "blue" | "red";
type Variant = `${Size}-${Color}`;
// `Variant` is 'small-blue' | 'small-red' | 'large-blue' | 'large-red'
```

This is extremely powerful for generating types for component variants, class names, or any string-based API that follows a consistent pattern.

### Intrinsic String Manipulation Types

TypeScript provides several utility types for manipulating strings at the type level, which are often used with template literals:

- `Capitalize<T>`: Converts the first character to uppercase.
- `Uncapitalize<T>`: Converts the first character to lowercase.
- `Uppercase<T>`: Converts the whole string to uppercase.
- `Lowercase<T>`: Converts the whole string to lowercase.

We used `Capitalize` in our example to convert `'top'` to `'Top'` to create `'marginTop'`.

### Production Perspective

- **CSS-in-JS Libraries**: This is the core mechanism behind modern type-safe CSS-in-JS libraries like Panda CSS, Stitches, and the styling engines in frameworks like Chakra UI. They use template literals and mapped types to generate thousands of type-safe style props based on your theme file.
- **Enforcing Design Tokens**: This pattern is the ultimate way to enforce the use of a design system's tokens (colors, spacing, fonts). It makes it easy for developers to do the right thing (use a theme value) and hard to do the wrong thing (use a magic number).
- **Type Generation, Not Runtime Logic**: Remember that all of this happens at the type level. It has zero runtime performance cost. It's purely a tool to improve the developer experience and prevent bugs during development.

## Mapped Types for Dynamic Props

## Learning Objective

Use mapped types to programmatically create new object types from unions or existing object keys, enabling dynamic and DRY prop definitions.

## Why This Matters

As your application and component library grow, you'll notice patterns in your prop types. You might have several components that all need a set of `margin` props, or you might want to create boolean variant props like `isPrimary` and `isSecondary`. Mapped types allow you to define this logic once and reuse it, keeping your types consistent and maintainable.

## Discovery Phase

Let's build a `Button` component that can have several boolean variant props, like `primary`, `secondary`, or `danger`.

A naive implementation would define them manually:

```javascript
type ButtonProps_Flawed = {
  primary?: boolean;
  secondary?: boolean;
  danger?: boolean;
  // What if we add a 'warning' variant? We have to update this manually.
};
```

This is not scalable. If we add a new variant, we have to remember to update this type. A mapped type lets us generate these props from a single source of truth.

```jsx
import React from 'react';

// Step 1: Define the single source of truth for our variants.
type ButtonVariant = 'primary' | 'secondary' | 'danger';

// Step 2: Use a mapped type to generate the boolean props.
// This reads: "For each key `V` in `ButtonVariant`, create a property
// where the key is `V` and the value is an optional boolean."
type VariantProps = {
  [V in ButtonVariant]?: boolean;
};
/*
  This generates the following type:
  {
    primary?: boolean;
    secondary?: boolean;
    danger?: boolean;
  }
*/

type ButtonProps = VariantProps & {
  children: React.ReactNode;
};

function Button({ children, ...variantProps }: ButtonProps) {
  let backgroundColor = 'grey';
  if (variantProps.primary) backgroundColor = 'blue';
  if (variantProps.secondary) backgroundColor = 'green';
  if (variantProps.danger) backgroundColor = 'red';

  return <button style={{ backgroundColor, color: 'white' }}>{children}</button>;
}

// --- USAGE ---
export default function App() {
  return (
    <div style={{ display: 'flex', gap: '10px' }}>
      <Button primary>Primary</Button>
      <Button secondary>Secondary</Button>
      <Button danger>Danger</Button>
    </div>
  );
}
```

**Rendered Output**:

```
[Primary] [Secondary] [Danger]
(Three buttons with blue, green, and red backgrounds respectively.)
```

Now, if we want to add a `'warning'` variant, we only need to change one line:
`type ButtonVariant = 'primary' | 'secondary' | 'danger' | 'warning';`
The `VariantProps` type will automatically update to include `warning?: boolean`. This is the power of DRY (Don't Repeat Yourself) for types.

## Deep Dive

### The Mapped Type Syntax

The syntax `[K in KeyType]: ValueType` is the core of mapped types. Let's break it down:

- **`K`**: A variable that represents the key in each iteration. You can name it anything.
- **`in`**: The keyword that iterates over a union of keys.
- **`KeyType`**: A union of string or number literals (e.g., `'primary' | 'secondary'`) or an object type whose keys we want to use (via `keyof T`).
- **`ValueType`**: The type of the value for each property. This can be a fixed type or can be computed based on `K`.

### Mapping Modifiers

You can add modifiers to control optionality (`?`) and writability (`readonly`).

```typescript
type MyType = {
  // Make all properties readonly
  readonly [K in "a" | "b"]: K;
};
// { readonly a: 'a'; readonly b: 'b'; }

type MyPartial<T> = {
  // Make all properties of T optional
  [K in keyof T]?: T[K];
};
// This is how the built-in `Partial<T>` utility type is implemented!
```

### Key Remapping with `as`

You can even change the names of the keys as you map them. This is often combined with template literal types.

```javascript
type Events = 'click' | 'focus';

// Let's create a props type for event handlers: `onClick`, `onFocus`
type EventProps = {
  // For each event `E` in `Events`, create a key `on${Capitalize<E>}`
  [E in Events as `on${Capitalize<E>}`]?: () => void;
};
/*
  Generates:
  {
    onClick?: () => void;
    onFocus?: () => void;
  }
*/
```

This advanced pattern allows you to generate entire APIs programmatically from a simple union of strings, which is incredibly powerful for building consistent and maintainable libraries.

### Production Perspective

- **Single Source of Truth**: Mapped types are essential for maintaining a single source of truth. Define your core concepts (like variants, sizes, colors) as simple union types, and then use mapped types to generate the complex prop types needed by your components.
- **Utility Types**: Many of TypeScript's built-in utility types (`Partial`, `Required`, `Readonly`, `Pick`, `Record`) are implemented using mapped types. Understanding mapped types helps you understand how these utilities work and even create your own custom utility types.
- **Code Generation at the Type Level**: Think of mapped types as a form of code generation that happens entirely within the type system. It lets you write more abstract and maintainable types that expand into the concrete definitions your components need.

## Type Guards and Narrowing

## Learning Objective

Write custom type guard functions to narrow down union types in complex scenarios where TypeScript's built-in analysis is not sufficient.

## Why This Matters

TypeScript's ability to narrow types in `if` and `switch` blocks is powerful, but it's based on simple checks (`typeof`, `instanceof`, property existence). Sometimes, your logic for determining a type is more complex. A custom type guard is a function you write that performs this complex check and explicitly tells the compiler, "If this function returns true, you can safely assume the variable is of this more specific type."

## Discovery Phase

Imagine we're fetching a list of `Media` items from an API. Some are `Video`s and some are `Image`s, and they have different properties.

```javascript
type Image = {
  kind: 'image';
  src: string;
  alt: string;
};

type Video = {
  kind: 'video';
  source: { format: 'mp4' | 'webm'; url: string };
  title: string;
};

type Media = Image | Video;

// Let's write a function to get the primary URL for any media item.
function getMediaUrl(media: Media): string {
  // TypeScript knows `media` is either Image or Video here.
  if (media.kind === 'image') {
    // This is standard narrowing. TS knows `media` is `Image`.
    return media.src;
  } else {
    // And here it knows `media` is `Video`.
    return media.source.url;
  }
}
```

This works great because we have a simple discriminant property, `kind`. But what if the data structure is less clean? What if we only have `src` for images and `source` for videos?

```jsx
type ImageWithNoKind = { src: string; alt: string };
type VideoWithNoKind = { source: { url: string }; title: string };
type MediaWithNoKind = ImageWithNoKind | VideoWithNoKind;

// Now, let's create a custom type guard function.
// The special return type `media is ImageWithNoKind` is the "type predicate".
function isImage(media: MediaWithNoKind): media is ImageWithNoKind {
  // Our custom logic: if it has an `src` property, it's an image.
  return 'src' in media;
}

function MediaDisplay({ media }: { media: MediaWithNoKind }) {
  // We use our type guard in an `if` statement.
  if (isImage(media)) {
    // Because `isImage` returned true, TypeScript narrows `media`
    // to `ImageWithNoKind` inside this block.
    return <img src={media.src} alt={media.alt} />;
  }

  // Outside the `if` block, TS knows `media` must be `VideoWithNoKind`.
  return <video src={media.source.url} title={media.title} />;
}
```

The function `isImage` is a custom type guard. Its special return type, `media is ImageWithNoKind`, is called a type predicate. It doesn't just return a boolean; it makes a type assertion to the compiler. This allows us to encapsulate complex checking logic into a reusable function that aids TypeScript's narrowing process.

## Deep Dive

### The Type Predicate

The syntax `argumentName is Type` is the key to a type guard.

- `argumentName` must be the name of one of the function's parameters.
- `Type` must be a more specific type than the parameter's original type.

The function must return a boolean. If it returns `true`, TypeScript will narrow the type of the argument to `Type` in the calling scope.

### When to Use Type Guards

Use a custom type guard when TypeScript's built-in narrowing isn't sufficient. This often happens when:

1.  **The check is complex**: The logic to differentiate types involves multiple properties or complex computations.
2.  **You want to reuse the logic**: The same type check is needed in multiple places. A type guard keeps your code DRY.
3.  **You're working with external data**: When data comes from an API, its shape might not be guaranteed. Type guards are a great way to validate and narrow the type of this data upon receipt.

### Production Perspective

- **Data Validation**: Type guards are a form of runtime data validation that also informs the static type system. They are a bridge between the unpredictable runtime world (e.g., API responses) and the statically-typed world of your application.
- **Assertion vs. Guard**: A related concept is a "type assertion function," which throws an error if the type is not what's expected.
  ```typescript
  function assertIsImage(media: Media): asserts media is Image {
    if (media.kind !== "image") throw new Error("Not an image!");
  }
  // After calling assertIsImage(myMedia), TS knows myMedia is an Image.
  ```
  Use guards for control flow (`if/else`) and assertions when an invalid type is an exceptional, unrecoverable error.
- **A Tool of Last Resort**: Always try to use simpler narrowing techniques first (like discriminated unions). Custom type guards are powerful, but they place the responsibility of correctness on you. If your `isImage` function has a bug and returns `true` for a video, TypeScript will trust you, and you'll get a runtime error.

## Branded Types for Domain Modeling

## Learning Objective

Use branded (or opaque) types to create distinct, non-interchangeable types for primitives like strings or numbers, preventing common domain logic errors.

## Why This Matters

In a large application, you might have many kinds of IDs: `UserID`, `ProductID`, `OrderID`. At the JavaScript level, these are all just strings or numbers. This makes it dangerously easy to mix them up, for example, by passing a `ProductID` to a function that fetches a user. Branded types solve this problem at the compile time, making such logical errors impossible.

## Discovery Phase

Let's see the problem in action with plain types.

```javascript
type UserID = string;
type ProductID = string;

function getUser(id: UserID) {
  console.log(`Fetching user with ID: ${id}`);
}

function getProduct(id: ProductID) {
  console.log(`Fetching product with ID: ${id}`);
}

const myUserId: UserID = 'user-123';
const myProductId: ProductID = 'prod-abc';

// The problem: TypeScript sees `UserID` and `ProductID` as just `string`.
// This is allowed, but it's a logical bug!
getUser(myProductId); // Prints "Fetching user with ID: prod-abc"
```

`type UserID = string` is just a type alias. It doesn't create a new, distinct type. To solve this, we can use a "brand"—a unique, non-existent property that we intersect with the primitive type.

```javascript
// Step 1: Define the branded types.
// We intersect `string` with an object that has a unique, private-like property.
export type UserID = string & { readonly __brand: 'UserID' };
export type ProductID = string & { readonly __brand: 'ProductID' };

// Step 2: Create "constructor" functions to safely create branded types.
// These are type assertion functions.
function asUserID(id: string): UserID {
  return id as UserID;
}
function asProductID(id: string): ProductID {
  return id as ProductID;
}

// Step 3: Update our functions to use the branded types.
function getUser(id: UserID) {
  console.log(`Fetching user with ID: ${id}`);
}
function getProduct(id: ProductID) {
  console.log(`Fetching product with ID: ${id}`);
}

// Step 4: Create instances using our constructors.
const myUserId = asUserID('user-123');
const myProductId = asProductID('prod-abc');

// Now, the logical bug is a compile-time error!
// getUser(myProductId);
// Error: Argument of type 'ProductID' is not assignable to parameter of type 'UserID'.
//        Type 'ProductID' is not assignable to type '{ readonly __brand: "UserID"; }'.

// This still works correctly.
getUser(myUserId);
```

This pattern is incredibly effective.

- At runtime, `myUserId` is still just the string `'user-123'`. There is zero performance overhead.
- At compile time, TypeScript sees `UserID` and `ProductID` as completely different and incompatible types because their `__brand` properties are different.

## Deep Dive

### How the Brand Works

The type `string & { readonly __brand: 'UserID' }` is an intersection. It describes a value that is _both_ a `string` and an object with that specific brand property. Of course, no such value can exist at runtime. But the type system doesn't know that. It uses this "impossible" type to create a unique "tag" or "brand" for our strings, allowing it to differentiate them.

The `as UserID` type assertion in our constructor function is where we tell the compiler, "Trust me, I know what I'm doing. This string should be treated as a `UserID` from now on."

### A Generic Brand Helper

You can create a generic helper type to make branding easier.

```javascript
type Brand<K, T> = K & { readonly __brand: T };

// Usage:
type Email = Brand<string, 'Email'>;
type CentAmount = Brand<number, 'CentAmount'>;

function asEmail(email: string): Email {
  // Add validation logic here!
  if (!email.includes('@')) throw new Error("Invalid email");
  return email as Email;
}

function sendEmail(to: Email) { /* ... */ }

const userEmail = asEmail('test@example.com');
sendEmail(userEmail); // OK

// const invalidEmail = 'not-an-email';
// sendEmail(invalidEmail); // Compile-time error!
```

This shows how branded types can be combined with validation logic in the constructor function to create highly secure and correct domain models.

### Production Perspective

- **Domain-Driven Design**: Branded types are a powerful tool for implementing Domain-Driven Design (DDD) in TypeScript. They allow you to create a rich, expressive type system that models your business domain and prevents invalid operations at the boundaries of your system.
- **API Safety**: Use branded types for any primitive value that has a specific meaning in your domain: IDs, emails, phone numbers, monetary amounts, etc. This is especially important in function signatures that form the public API of your modules.
- **Boilerplate vs. Safety**: The main drawback of this pattern is the boilerplate of creating the brand and the constructor function. This is a trade-off. For simple applications, it might be overkill. For large, complex systems where logical correctness is critical (e.g., financial tech, healthcare), the safety it provides is well worth the extra code.

## Module Synthesis 📋

## Module Synthesis

In this chapter, we have ventured into the most advanced and powerful corners of TypeScript as applied to React. We've moved beyond typing simple props and state to crafting component APIs that are not just safe, but also flexible, expressive, and highly reusable.

We started with **Discriminated Unions**, a robust pattern for creating components with mutually exclusive modes, eliminating impossible prop combinations. We then laid the conceptual groundwork for **Polymorphic Components** before implementing the industry-standard **"as prop" pattern**, enabling us to build foundational design system components that can render as any HTML element while preserving type safety.

We explored how to model intricate prop dependencies with **Conditional Types**, creating "smart" components that adapt their types based on their props. We then saw how **Template Literal Types** and **Mapped Types** work together to programmatically generate entire sets of props from a single source of truth, keeping our design systems DRY and consistent.

Finally, we looked at two patterns for enhancing runtime correctness through the type system. **Custom Type Guards** allow us to give hints to the compiler in complex narrowing scenarios, while **Branded Types** provide a way to create distinct types for primitives, preventing critical domain logic errors.

## Looking Ahead

You are now equipped with the patterns used by professional developers to build world-class component libraries and design systems. You can create APIs that are not only safe but also provide an exceptional developer experience.

In the next chapter, **Chapter 23: Typing Hooks and State**, we will shift our focus inward. We'll take these advanced concepts and apply them to the internal logic of our components. We'll cover advanced patterns for typing `useState`, mastering the complex types of `useReducer`, ensuring type-safe global state with `useContext`, and creating fully-typed, reusable custom hooks.
