# Chapter Appendix A: Essential VS Code Setup

## The Editor as Diagnostic Partner

## The Editor as Diagnostic Partner

Your code editor is not just a text input device—it's your first line of defense against bugs, your performance monitor, and your documentation assistant. A properly configured VS Code setup catches errors before they reach the browser, suggests fixes before you ask, and surfaces problems the moment they're introduced.

This appendix establishes a reference VS Code configuration that evolves through progressive enhancement. We'll start with a minimal setup that works, expose its limitations through concrete failures, and systematically add tooling that solves each problem.

### The Reference Implementation: A React/Next.js Project

Throughout this appendix, we'll configure VS Code for a realistic Next.js project structure:

**Project Structure**:
```
my-app/
├── src/
│   ├── app/
│   │   ├── page.tsx
│   │   └── layout.tsx
│   ├── components/
│   │   ├── UserProfile.tsx
│   │   └── ProductCard.tsx
│   └── lib/
│       └── utils.ts
├── public/
├── package.json
├── tsconfig.json
└── next.config.js
```

This structure will serve as our testing ground for each configuration improvement.

## Phase 1: The Baseline - Vanilla VS Code

## Phase 1: The Baseline - Vanilla VS Code

### Iteration 0: Fresh VS Code Installation

Let's start with a completely fresh VS Code installation and a simple React component to see what problems emerge.

```tsx
// src/components/UserProfile.tsx
export default function UserProfile({ userId }) {
  const [user, setUser] = useState();
  
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data));
  }, []);
  
  return (
    <div className="profile">
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### The Failure: Silent Problems Everywhere

**VS Code Behavior**:
- No red squiggles under any code
- No warnings in the Problems panel
- File saves successfully
- No indication anything is wrong

**Browser Console Output** (after running the app):
```
Uncaught ReferenceError: useState is not defined
    at UserProfile (UserProfile.tsx:2)

Warning: React Hook useEffect is called in a function that is neither a React function component nor a custom React Hook function.

Uncaught TypeError: Cannot read properties of undefined (reading 'name')
    at UserProfile (UserProfile.tsx:13)
```

**Terminal Output**:
```bash
Type error: Parameter 'userId' implicitly has an 'any' type.

Type error: Cannot find name 'useState'.

Type error: Cannot find name 'useEffect'.
```

### Diagnostic Analysis: Reading the Silence

**What the editor shows**: Nothing. The code looks fine in VS Code.

**What the browser reveals**: Multiple runtime errors that should have been caught during development.

**What the terminal reveals**: TypeScript compilation fails, but only when you try to build.

**Root cause identified**: VS Code has no understanding of:
1. React imports (missing `import` statements)
2. TypeScript types (no type checking active)
3. React-specific patterns (hooks, JSX)

**Why vanilla VS Code can't solve this**: Without extensions and configuration, VS Code treats `.tsx` files as plain text with syntax highlighting. It has no semantic understanding of React or TypeScript.

**What we need**: Extensions that provide language intelligence, type checking, and React-specific linting.

## Phase 2: Essential Extensions

## Phase 2: Essential Extensions

### Iteration 1: Installing Core Extensions

The following extensions transform VS Code from a text editor into a React development environment.

### Extension 1: ESLint - The Error Detector

**What it does**: Analyzes your code for problems in real-time, enforcing code quality rules and catching common mistakes.

**Installation**:
1. Open VS Code Extensions panel (`Cmd/Ctrl + Shift + X`)
2. Search for "ESLint" (by Microsoft)
3. Click Install
4. Reload VS Code

**Project Setup**:

```bash
# Install ESLint and React-specific plugins
npm install --save-dev eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
npm install --save-dev eslint-plugin-react eslint-plugin-react-hooks
npm install --save-dev eslint-config-next  # For Next.js projects
```

**Configuration File**:

```json
// .eslintrc.json
{
  "extends": [
    "next/core-web-vitals",
    "plugin:@typescript-eslint/recommended",
    "plugin:react-hooks/recommended"
  ],
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": "latest",
    "sourceType": "module",
    "ecmaFeatures": {
      "jsx": true
    }
  },
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn",
    "@typescript-eslint/no-unused-vars": "warn",
    "@typescript-eslint/no-explicit-any": "warn"
  }
}
```

**VS Code Settings** (add to `.vscode/settings.json`):

```json
{
  "eslint.validate": [
    "javascript",
    "javascriptreact",
    "typescript",
    "typescriptreact"
  ],
  "eslint.run": "onType"
}
```

### The Improvement: Immediate Error Detection

Now when we open `UserProfile.tsx`:

**VS Code Behavior**:
- Red squiggles appear under `useState` and `useEffect`
- Problems panel shows: `'useState' is not defined. eslint(no-undef)`
- Hover over `userId` shows: `Parameter 'userId' implicitly has an 'any' type.`
- Red squiggles under `user.name` and `user.email`

**Expected vs. Actual**:
- **Before**: No indication of problems until runtime
- **After**: 6 errors highlighted immediately in the editor

The editor now catches the missing imports and type errors before we even save the file.

### Extension 2: Prettier - The Formatter

**What it does**: Automatically formats code on save, eliminating formatting debates and ensuring consistency.

**Installation**:

```bash
npm install --save-dev prettier eslint-config-prettier
```

**Configuration File**:

```json
// .prettierrc
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": false,
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false
}
```

**VS Code Settings** (add to `.vscode/settings.json`):

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

**Update ESLint config** to prevent conflicts:

```json
// .eslintrc.json
{
  "extends": [
    "next/core-web-vitals",
    "plugin:@typescript-eslint/recommended",
    "plugin:react-hooks/recommended",
    "prettier"  // ← Add this last to disable conflicting rules
  ]
}
```

### The Improvement: Consistent Formatting

**Before**:

```tsx
// Inconsistent formatting
export default function UserProfile({userId}:{userId:string}) {
const [user,setUser]=useState<User|null>(null)
  useEffect(()=>{
fetch(`/api/users/${userId}`).then(res=>res.json()).then(data=>setUser(data))
},[userId])
return <div className="profile"><h1>{user?.name}</h1></div>
}
```

**After** (automatic on save):

```tsx
// Consistently formatted
export default function UserProfile({
  userId,
}: {
  userId: string;
}) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then((data) => setUser(data));
  }, [userId]);

  return (
    <div className="profile">
      <h1>{user?.name}</h1>
    </div>
  );
}
```

### Extension 3: ES7+ React/Redux/React-Native Snippets

**What it does**: Provides shortcuts for common React patterns, reducing boilerplate typing.

**Installation**: Search for "ES7+ React/Redux/React-Native snippets" in Extensions panel.

**Key Snippets**:

| Trigger | Expands To |
|---------|------------|
| `rafce` | React Arrow Function Component Export |
| `useState` | `const [state, setState] = useState(initialValue)` |
| `useEffect` | `useEffect(() => { }, [])` |
| `useMemo` | `useMemo(() => { }, [])` |
| `useCallback` | `useCallback(() => { }, [])` |

**Example Usage**:

Type `rafce` and press Tab:

```tsx
// Expands to:
import React from "react";

const ComponentName = () => {
  return <div>ComponentName</div>;
};

export default ComponentName;
```

### Extension 4: Error Lens - Inline Error Display

**What it does**: Shows error messages directly in the editor line, not just in the Problems panel.

**Installation**: Search for "Error Lens" in Extensions panel.

**VS Code Settings**:

```json
{
  "errorLens.enabledDiagnosticLevels": ["error", "warning"],
  "errorLens.fontSize": "0.9em",
  "errorLens.fontWeight": "normal"
}
```

### The Improvement: Errors You Can't Miss

**Before** (standard VS Code):
- Red squiggle under `user.name`
- Must hover to see: "Object is possibly 'undefined'"

**After** (with Error Lens):
- Red squiggle under `user.name`
- Error message displayed inline: `Object is possibly 'undefined'. ts(2532)`
- No hovering required

### Extension 5: Auto Rename Tag

**What it does**: Automatically renames paired HTML/JSX tags when you edit one.

**Installation**: Search for "Auto Rename Tag" in Extensions panel.

**Demonstration**:

Change `<div>` to `<section>`:

## Phase 3: TypeScript Configuration

## Phase 3: TypeScript Configuration

### Iteration 2: Enabling Strict Type Checking

With extensions installed, let's configure TypeScript for maximum safety.

### The Failure: Loose Type Checking Misses Bugs

**Current tsconfig.json** (Next.js default):

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": false,  // ← Problem: too permissive
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

**Problem Code** (compiles without errors):

```tsx
// src/components/ProductCard.tsx
export default function ProductCard({ product }) {  // ← No type error
  return (
    <div>
      <h2>{product.name}</h2>
      <p>${product.price}</p>
      <button onClick={() => addToCart(product.id)}>Add to Cart</button>
    </div>
  );
}

function addToCart(id) {  // ← No type error
  // Implementation
}
```

**Browser Console Output**:
```
Uncaught TypeError: Cannot read properties of undefined (reading 'name')
    at ProductCard (ProductCard.tsx:4)
```

**Diagnostic Analysis**:

**What TypeScript shows**: No errors. Code compiles successfully.

**What the browser reveals**: Runtime error when `product` is undefined.

**Root cause**: `strict: false` allows implicit `any` types, defeating TypeScript's purpose.

**Why loose checking can't solve this**: Without strict mode, TypeScript can't enforce type safety on function parameters, return values, or null checks.

**What we need**: Strict TypeScript configuration that catches these issues at compile time.

### The Solution: Strict TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,  // ← Enable all strict checks
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    
    // Additional strict checks
    "noUncheckedIndexedAccess": true,  // Array access returns T | undefined
    "noImplicitReturns": true,  // All code paths must return a value
    "noFallthroughCasesInSwitch": true,  // Switch cases must break/return
    "forceConsistentCasingInFileNames": true,  // Prevent import casing issues
    
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### The Improvement: Compile-Time Error Detection

**VS Code Behavior** (after saving tsconfig.json):
- Red squiggle under `product` parameter: `Parameter 'product' implicitly has an 'any' type.`
- Red squiggle under `id` parameter: `Parameter 'id' implicitly has an 'any' type.`
- Red squiggle under `product.name`: `'product' is of type 'unknown'.`

**Fixed Code**:

```tsx
// src/components/ProductCard.tsx
interface Product {
  id: string;
  name: string;
  price: number;
}

interface ProductCardProps {
  product: Product;
}

export default function ProductCard({ product }: ProductCardProps) {
  return (
    <div>
      <h2>{product.name}</h2>
      <p>${product.price}</p>
      <button onClick={() => addToCart(product.id)}>Add to Cart</button>
    </div>
  );
}

function addToCart(id: string): void {
  // Implementation
}
```

**Expected vs. Actual**:
- **Before**: Runtime error when product is undefined
- **After**: Compile-time error prevents the bug from reaching the browser

### VS Code TypeScript Settings

```json
// .vscode/settings.json
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "typescript.preferences.importModuleSpecifier": "relative",
  "typescript.updateImportsOnFileMove.enabled": "always",
  "typescript.suggest.autoImports": true,
  "typescript.preferences.includePackageJsonAutoImports": "on"
}
```

### When to Apply Strict TypeScript

**What it optimizes for**:
- Compile-time error detection
- Refactoring safety
- Self-documenting code

**What it sacrifices**:
- Initial development speed (must write types)
- Flexibility with dynamic data

**When to choose this approach**:
- New projects (start strict from day one)
- Production applications
- Team environments (prevents type-related bugs)

**When to avoid this approach**:
- Rapid prototyping (use `strict: false` temporarily)
- Migrating large JavaScript codebases (enable gradually)

## Phase 4: Workspace Settings

## Phase 4: Workspace Settings

### Iteration 3: Project-Specific Configuration

Create a `.vscode/settings.json` file in your project root to ensure consistent settings across your team.

### The Complete VS Code Settings File

```json
// .vscode/settings.json
{
  // Editor Behavior
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit",
    "source.organizeImports": "explicit"
  },
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.detectIndentation": false,
  
  // File Management
  "files.autoSave": "onFocusChange",
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.exclude": {
    "**/.next": true,
    "**/node_modules": true,
    "**/.git": true
  },
  
  // TypeScript
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "typescript.preferences.importModuleSpecifier": "relative",
  "typescript.updateImportsOnFileMove.enabled": "always",
  "typescript.suggest.autoImports": true,
  "typescript.preferences.includePackageJsonAutoImports": "on",
  
  // ESLint
  "eslint.validate": [
    "javascript",
    "javascriptreact",
    "typescript",
    "typescriptreact"
  ],
  "eslint.run": "onType",
  
  // Language-Specific Formatting
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[json]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[jsonc]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  
  // Tailwind CSS (if using)
  "tailwindCSS.experimental.classRegex": [
    ["cva\\(([^)]*)\\)", "[\"'`]([^\"'`]*).*?[\"'`]"],
    ["cn\\(([^)]*)\\)", "(?:'|\"|`)([^']*)(?:'|\"|`)"]
  ],
  
  // Error Lens
  "errorLens.enabledDiagnosticLevels": ["error", "warning"],
  "errorLens.fontSize": "0.9em",
  
  // Search
  "search.exclude": {
    "**/node_modules": true,
    "**/.next": true,
    "**/dist": true,
    "**/build": true
  }
}
```

### The Improvement: Consistent Team Experience

**Before**:
- Developer A uses tabs, Developer B uses spaces
- Developer C's editor doesn't auto-format
- Import organization differs between developers

**After**:
- All developers use the same formatting
- Code is auto-formatted on save
- Imports are automatically organized
- ESLint errors are fixed automatically when possible

### Recommended Extensions File

Create `.vscode/extensions.json` to suggest extensions to team members:

```json
// .vscode/extensions.json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "dsznajder.es7-react-js-snippets",
    "usernamehw.errorlens",
    "formulahendry.auto-rename-tag",
    "bradlc.vscode-tailwindcss"
  ]
}
```

When a team member opens the project, VS Code will prompt them to install the recommended extensions.

## Phase 5: Keyboard Shortcuts and Productivity

## Phase 5: Keyboard Shortcuts and Productivity

### Essential Keyboard Shortcuts

Master these shortcuts to dramatically increase development speed:

### Navigation

| Shortcut | Action | Use Case |
|----------|--------|----------|
| `Cmd/Ctrl + P` | Quick Open | Jump to any file by name |
| `Cmd/Ctrl + Shift + O` | Go to Symbol | Jump to function/component in current file |
| `Cmd/Ctrl + T` | Go to Symbol in Workspace | Find any function across all files |
| `F12` | Go to Definition | Jump to where a function/component is defined |
| `Shift + F12` | Find All References | See everywhere a function is used |
| `Cmd/Ctrl + Click` | Go to Definition | Alternative to F12 |
| `Cmd/Ctrl + -` | Go Back | Return to previous cursor position |
| `Cmd/Ctrl + Shift + -` | Go Forward | Opposite of Go Back |

### Editing

| Shortcut | Action | Use Case |
|----------|--------|----------|
| `Cmd/Ctrl + D` | Select Next Occurrence | Multi-cursor editing |
| `Cmd/Ctrl + Shift + L` | Select All Occurrences | Select all instances of current word |
| `Alt + Click` | Add Cursor | Multiple cursors at arbitrary positions |
| `Cmd/Ctrl + /` | Toggle Line Comment | Comment/uncomment code |
| `Shift + Alt + A` | Toggle Block Comment | Multi-line comment |
| `Alt + Up/Down` | Move Line Up/Down | Reorder code |
| `Shift + Alt + Up/Down` | Copy Line Up/Down | Duplicate lines |
| `Cmd/Ctrl + Shift + K` | Delete Line | Remove entire line |

### Code Actions

| Shortcut | Action | Use Case |
|----------|--------|----------|
| `Cmd/Ctrl + .` | Quick Fix | Show available fixes for errors |
| `F2` | Rename Symbol | Rename variable/function everywhere |
| `Cmd/Ctrl + Shift + R` | Refactor | Show refactoring options |
| `Cmd/Ctrl + Space` | Trigger Suggest | Show autocomplete |
| `Cmd/Ctrl + Shift + Space` | Trigger Parameter Hints | Show function signature |

### Search and Replace

| Shortcut | Action | Use Case |
|----------|--------|----------|
| `Cmd/Ctrl + F` | Find | Search in current file |
| `Cmd/Ctrl + H` | Replace | Find and replace in current file |
| `Cmd/Ctrl + Shift + F` | Find in Files | Search across entire project |
| `Cmd/Ctrl + Shift + H` | Replace in Files | Replace across entire project |

### Panel Management

| Shortcut | Action | Use Case |
|----------|--------|----------|
| `Cmd/Ctrl + B` | Toggle Sidebar | Show/hide file explorer |
| `Cmd/Ctrl + J` | Toggle Panel | Show/hide terminal/problems/output |
| `` Cmd/Ctrl + ` `` | Toggle Terminal | Quick terminal access |
| `Cmd/Ctrl + Shift + E` | Focus Explorer | Jump to file explorer |
| `Cmd/Ctrl + Shift + M` | Focus Problems | Jump to problems panel |

### Custom Keybindings

Add these to your `keybindings.json` (Cmd/Ctrl + K, Cmd/Ctrl + S):

```json
// keybindings.json
[
  {
    "key": "cmd+shift+7",
    "command": "editor.action.commentLine",
    "when": "editorTextFocus && !editorReadonly"
  },
  {
    "key": "cmd+k cmd+c",
    "command": "editor.action.addCommentLine",
    "when": "editorTextFocus && !editorReadonly"
  },
  {
    "key": "cmd+k cmd+u",
    "command": "editor.action.removeCommentLine",
    "when": "editorTextFocus && !editorReadonly"
  }
]
```

## Phase 6: Debugging Configuration

## Phase 6: Debugging Configuration

### Iteration 4: Setting Up the Debugger

VS Code's built-in debugger is more powerful than `console.log` debugging. Let's configure it for Next.js.

### The Failure: Console.log Debugging at Scale

**Current Debugging Approach**:

```tsx
// src/components/UserProfile.tsx
export default function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  
  console.log("UserProfile rendered, userId:", userId);  // ← Manual logging
  
  useEffect(() => {
    console.log("Effect running, fetching user:", userId);  // ← Manual logging
    
    fetch(`/api/users/${userId}`)
      .then((res) => {
        console.log("Response received:", res);  // ← Manual logging
        return res.json();
      })
      .then((data) => {
        console.log("Data parsed:", data);  // ← Manual logging
        setUser(data);
      });
  }, [userId]);
  
  console.log("Current user state:", user);  // ← Manual logging
  
  return (
    <div>
      <h1>{user?.name}</h1>
    </div>
  );
}
```

**Problems**:
1. Must manually add/remove console.log statements
2. Console output becomes cluttered with multiple components
3. Can't inspect variable state at arbitrary points
4. Must rebuild to see changes
5. Console.log statements often left in production code

### The Solution: VS Code Debugger Configuration

Create `.vscode/launch.json`:

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Next.js: debug server-side",
      "type": "node-terminal",
      "request": "launch",
      "command": "npm run dev"
    },
    {
      "name": "Next.js: debug client-side",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:3000",
      "webRoot": "${workspaceFolder}"
    },
    {
      "name": "Next.js: debug full stack",
      "type": "node-terminal",
      "request": "launch",
      "command": "npm run dev",
      "serverReadyAction": {
        "pattern": "- Local:.+(https?://.+)",
        "uriFormat": "%s",
        "action": "debugWithChrome"
      }
    }
  ]
}
```

### Using the Debugger

**Step 1: Set Breakpoints**

Click in the gutter (left of line numbers) to set a breakpoint:

```tsx
// src/components/UserProfile.tsx
export default function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    // ← Click here to set breakpoint (red dot appears)
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then((data) => {
        // ← Click here to set another breakpoint
        setUser(data);
      });
  }, [userId]);
  
  return (
    <div>
      <h1>{user?.name}</h1>
    </div>
  );
}
```

**Step 2: Start Debugging**

1. Press `F5` or click "Run and Debug" in sidebar
2. Select "Next.js: debug full stack"
3. Browser opens automatically
4. Navigate to page with UserProfile component

**Step 3: Inspect Variables**

When breakpoint hits:
- **Variables panel**: Shows all local variables and their values
- **Watch panel**: Add expressions to monitor (e.g., `user?.name`)
- **Call Stack panel**: Shows function call hierarchy
- **Debug Console**: Execute arbitrary code in current context

**Step 4: Control Execution**

| Button | Shortcut | Action |
|--------|----------|--------|
| Continue | `F5` | Resume execution until next breakpoint |
| Step Over | `F10` | Execute current line, don't enter functions |
| Step Into | `F11` | Enter function calls |
| Step Out | `Shift + F11` | Exit current function |
| Restart | `Cmd/Ctrl + Shift + F5` | Restart debugging session |
| Stop | `Shift + F5` | Stop debugging |

### The Improvement: Precise Debugging

**Before** (console.log):
```
UserProfile rendered, userId: 123
Effect running, fetching user: 123
Response received: Response { ... }
Data parsed: { id: "123", name: "John", email: "john@example.com" }
Current user state: { id: "123", name: "John", email: "john@example.com" }
```

**After** (debugger):
- Execution pauses at exact line
- Inspect `userId`, `res`, `data`, `user` in Variables panel
- Evaluate expressions in Debug Console: `data.name.toUpperCase()`
- See call stack: `UserProfile → useEffect → fetch.then`
- No console clutter
- No manual logging code

### Conditional Breakpoints

Right-click on a breakpoint to add a condition:

```tsx
// Only break when userId is "123"
useEffect(() => {
  // Breakpoint condition: userId === "123"
  fetch(`/api/users/${userId}`)
    .then((res) => res.json())
    .then((data) => setUser(data));
}, [userId]);
```

### Logpoints (Non-Breaking Logging)

Right-click in gutter → "Add Logpoint":

```tsx
// Logpoint: "Fetching user {userId}"
useEffect(() => {
  fetch(`/api/users/${userId}`)  // ← Logpoint here
    .then((res) => res.json())
    .then((data) => setUser(data));
}, [userId]);
```

**Output** (in Debug Console):
```
Fetching user 123
Fetching user 456
```

No code changes required. Logpoints don't pause execution.

## Phase 7: Snippets and Code Generation

## Phase 7: Snippets and Code Generation

### Iteration 5: Custom Snippets for React Patterns

Create custom snippets for patterns you use frequently.

### Creating Custom Snippets

1. Open Command Palette (`Cmd/Ctrl + Shift + P`)
2. Type "Configure User Snippets"
3. Select "typescriptreact.json"

### Essential React Snippets

```json
// typescriptreact.json
{
  "React Function Component with TypeScript": {
    "prefix": "rfc",
    "body": [
      "interface ${1:ComponentName}Props {",
      "  $2",
      "}",
      "",
      "export default function ${1:ComponentName}({ $3 }: ${1:ComponentName}Props) {",
      "  return (",
      "    <div>",
      "      $0",
      "    </div>",
      "  );",
      "}"
    ],
    "description": "React Function Component with TypeScript"
  },
  
  "useState Hook": {
    "prefix": "us",
    "body": [
      "const [${1:state}, set${1/(.*)/${1:/capitalize}/}] = useState<${2:type}>(${3:initialValue});"
    ],
    "description": "useState hook with TypeScript"
  },
  
  "useEffect Hook": {
    "prefix": "ue",
    "body": [
      "useEffect(() => {",
      "  $1",
      "  ",
      "  return () => {",
      "    $2",
      "  };",
      "}, [$3]);"
    ],
    "description": "useEffect with cleanup"
  },
  
  "React Query useQuery": {
    "prefix": "uq",
    "body": [
      "const { data: ${1:data}, isLoading, error } = useQuery({",
      "  queryKey: [${2:'key'}],",
      "  queryFn: async () => {",
      "    const res = await fetch(${3:url});",
      "    if (!res.ok) throw new Error('Failed to fetch');",
      "    return res.json();",
      "  },",
      "});"
    ],
    "description": "React Query useQuery hook"
  },
  
  "React Query useMutation": {
    "prefix": "um",
    "body": [
      "const ${1:mutation} = useMutation({",
      "  mutationFn: async (${2:data}: ${3:Type}) => {",
      "    const res = await fetch(${4:url}, {",
      "      method: 'POST',",
      "      headers: { 'Content-Type': 'application/json' },",
      "      body: JSON.stringify(${2:data}),",
      "    });",
      "    if (!res.ok) throw new Error('Failed to mutate');",
      "    return res.json();",
      "  },",
      "  onSuccess: () => {",
      "    $5",
      "  },",
      "});"
    ],
    "description": "React Query useMutation hook"
  },
  
  "Zustand Store": {
    "prefix": "zstore",
    "body": [
      "interface ${1:Store}State {",
      "  ${2:value}: ${3:type};",
      "  set${2/(.*)/${1:/capitalize}/}: (${2:value}: ${3:type}) => void;",
      "}",
      "",
      "export const use${1:Store} = create<${1:Store}State>((set) => ({",
      "  ${2:value}: ${4:initialValue},",
      "  set${2/(.*)/${1:/capitalize}/}: (${2:value}) => set({ ${2:value} }),",
      "}));"
    ],
    "description": "Zustand store with TypeScript"
  },
  
  "Next.js Server Component": {
    "prefix": "nsc",
    "body": [
      "interface ${1:ComponentName}Props {",
      "  $2",
      "}",
      "",
      "export default async function ${1:ComponentName}({ $3 }: ${1:ComponentName}Props) {",
      "  const data = await fetch(${4:url}).then(res => res.json());",
      "  ",
      "  return (",
      "    <div>",
      "      $0",
      "    </div>",
      "  );",
      "}"
    ],
    "description": "Next.js Server Component"
  },
  
  "Next.js API Route": {
    "prefix": "napi",
    "body": [
      "import { NextRequest, NextResponse } from 'next/server';",
      "",
      "export async function ${1|GET,POST,PUT,DELETE,PATCH|}(request: NextRequest) {",
      "  try {",
      "    $2",
      "    ",
      "    return NextResponse.json({ $3 });",
      "  } catch (error) {",
      "    return NextResponse.json(",
      "      { error: 'Internal Server Error' },",
      "      { status: 500 }",
      "    );",
      "  }",
      "}"
    ],
    "description": "Next.js API Route Handler"
  },
  
  "Try-Catch Block": {
    "prefix": "tryc",
    "body": [
      "try {",
      "  $1",
      "} catch (error) {",
      "  console.error('${2:Error message}:', error);",
      "  $3",
      "}"
    ],
    "description": "Try-catch block"
  },
  
  "Console Log": {
    "prefix": "cl",
    "body": [
      "console.log('${1:label}:', $2);"
    ],
    "description": "Console log with label"
  }
}
```

### Using Snippets

Type the prefix and press `Tab`:

**Example 1: Create a component**
```
rfc<Tab>
```

Expands to:

```tsx
interface ComponentNameProps {
  // cursor here
}

export default function ComponentName({ }: ComponentNameProps) {
  return (
    <div>
      // cursor moves here after Tab
    </div>
  );
}
```

**Example 2: Add useState**
```
us<Tab>
```

Expands to:

```tsx
const [state, setState] = useState<type>(initialValue);
//     ^cursor here, type name, press Tab to move to next placeholder
```

### Snippet Variables

Snippets support dynamic variables:

| Variable | Description |
|----------|-------------|
| `$1`, `$2`, `$3` | Tab stops (cursor positions) |
| `$0` | Final cursor position |
| `${1:placeholder}` | Tab stop with default text |
| `${1\|option1,option2\|}` | Dropdown selection |
| `${1/(.*)/${1:/capitalize}/}` | Transform text (capitalize, etc.) |

### The Improvement: Faster Boilerplate

**Before** (manual typing):
- Type entire component structure
- Remember TypeScript interface syntax
- Type prop destructuring
- 2-3 minutes per component

**After** (with snippets):
- Type `rfc<Tab>`
- Fill in component name
- Fill in props
- 10-15 seconds per component

**Time saved**: ~80% reduction in boilerplate typing

## Phase 8: Git Integration

## Phase 8: Git Integration

### Built-in Git Features

VS Code has excellent Git integration without additional extensions.

### Source Control Panel

**Access**: Click Source Control icon in sidebar or press `Cmd/Ctrl + Shift + G`

**Features**:
- View changed files
- Stage/unstage changes
- Commit with message
- Push/pull
- View diff inline

### Git Commands in Command Palette

Press `Cmd/Ctrl + Shift + P` and type "Git":

| Command | Action |
|---------|--------|
| `Git: Commit` | Commit staged changes |
| `Git: Push` | Push to remote |
| `Git: Pull` | Pull from remote |
| `Git: Checkout to...` | Switch branches |
| `Git: Create Branch...` | Create new branch |
| `Git: Merge Branch...` | Merge another branch |
| `Git: Stash` | Stash changes |
| `Git: Unstash` | Apply stashed changes |

### Inline Git Blame (Optional: GitLens Extension)

If you want to see who changed each line:

**Installation**: Search for "GitLens" in Extensions panel.

**Features**:
- Inline blame annotations
- File history
- Line history
- Compare branches
- Rich commit details

**VS Code Settings** (to reduce clutter):

```json
{
  "gitlens.currentLine.enabled": false,
  "gitlens.hovers.currentLine.over": "line",
  "gitlens.codeLens.enabled": false
}
```

### Git Ignore

Create `.gitignore` in project root:

```bash
# .gitignore
# Dependencies
node_modules/
.pnp
.pnp.js

# Next.js
.next/
out/
build/
dist/

# Environment variables
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# Debug
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# IDE
.vscode/*
!.vscode/settings.json
!.vscode/tasks.json
!.vscode/launch.json
!.vscode/extensions.json
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Testing
coverage/
.nyc_output/

# Misc
*.log
.cache/
```

**Note**: We include `.vscode/settings.json` in Git so team members share the same configuration, but exclude personal settings like window state.

## Phase 9: Performance and Bundle Analysis

## Phase 9: Performance and Bundle Analysis

### Extension: Import Cost

**What it does**: Shows the size of imported packages inline.

**Installation**: Search for "Import Cost" in Extensions panel.

**Example**:

```tsx
// Shows size next to import
import { format } from "date-fns";  // 📦 12.5 KB (gzipped: 4.2 KB)
import { Button } from "@/components/ui/button";  // 📦 2.1 KB (gzipped: 0.8 KB)
import _ from "lodash";  // 📦 72.4 KB (gzipped: 24.8 KB) ⚠️
```

**Diagnostic Value**: Immediately see when you're importing a large library. In the example above, the full lodash import is a red flag—use `lodash-es` or import specific functions instead.

### Bundle Analysis

Add to `package.json`:

```json
{
  "scripts": {
    "analyze": "ANALYZE=true next build"
  },
  "devDependencies": {
    "@next/bundle-analyzer": "^14.0.0"
  }
}
```

Create `next.config.js`:

```javascript
// next.config.js
const withBundleAnalyzer = require("@next/bundle-analyzer")({
  enabled: process.env.ANALYZE === "true",
});

module.exports = withBundleAnalyzer({
  // Your Next.js config
});
```

**Run analysis**:

```bash
npm run analyze
```

Opens interactive treemap showing bundle composition. Use this to identify:
- Large dependencies that could be replaced
- Duplicate packages
- Unused code that should be removed

## The Complete Setup Checklist

## The Complete Setup Checklist

### Initial Setup (One-Time)

**Extensions** (install these):
- [ ] ESLint (Microsoft)
- [ ] Prettier - Code formatter (Prettier)
- [ ] ES7+ React/Redux/React-Native snippets (dsznajder)
- [ ] Error Lens (Alexander)
- [ ] Auto Rename Tag (Jun Han)
- [ ] Tailwind CSS IntelliSense (Tailwind Labs) - if using Tailwind
- [ ] Import Cost (Wix) - optional but recommended
- [ ] GitLens (GitKraken) - optional

**Project Files** (create these):
- [ ] `.vscode/settings.json` - Workspace settings
- [ ] `.vscode/extensions.json` - Recommended extensions
- [ ] `.vscode/launch.json` - Debugger configuration
- [ ] `.eslintrc.json` - ESLint configuration
- [ ] `.prettierrc` - Prettier configuration
- [ ] `tsconfig.json` - TypeScript configuration (strict mode)
- [ ] `.gitignore` - Git ignore rules

**VS Code Settings** (configure these):
- [ ] Format on save enabled
- [ ] ESLint auto-fix on save enabled
- [ ] TypeScript strict mode enabled
- [ ] Auto-import enabled
- [ ] Organize imports on save enabled

**Custom Snippets** (create these):
- [ ] React component snippet (`rfc`)
- [ ] useState snippet (`us`)
- [ ] useEffect snippet (`ue`)
- [ ] React Query snippets (`uq`, `um`)
- [ ] Zustand store snippet (`zstore`)

### Per-Project Setup (Each New Project)

- [ ] Copy `.vscode/` folder from template
- [ ] Install dependencies: `npm install`
- [ ] Verify ESLint works: Open a `.tsx` file, introduce an error, see red squiggle
- [ ] Verify Prettier works: Save a file, see it auto-format
- [ ] Verify TypeScript works: Remove a type, see error
- [ ] Set up debugger: Press F5, verify it launches
- [ ] Test snippets: Type `rfc<Tab>`, verify it expands

### Verification Tests

**Test 1: ESLint Detection**

```tsx
// Create this file and verify you see errors
export default function Test() {
  const [count, setCount] = useState(0);  // ← Should show error: useState not imported
  
  useEffect(() => {
    console.log(count);
  });  // ← Should show warning: missing dependency array
  
  return <div>{count}</div>;
}
```

**Expected**: 2 errors/warnings visible immediately.

**Test 2: Auto-Format**

```tsx
// Type this messily and save
export default function Test(){const x=1;return <div>{x}</div>}
```

**Expected**: Auto-formats to:

```tsx
export default function Test() {
  const x = 1;
  return <div>{x}</div>;
}
```

**Test 3: TypeScript Checking**

```tsx
// Create this and verify you see type error
interface Props {
  name: string;
}

export default function Test({ name }: Props) {
  return <div>{name.toUpperCase()}</div>;
}

// Usage - should show error
<Test name={123} />  // ← Type error: number not assignable to string
```

**Expected**: Red squiggle under `name={123}`.

**Test 4: Debugger**

1. Set breakpoint in any component
2. Press F5
3. Navigate to that component in browser
4. Verify execution pauses at breakpoint

**Expected**: Debugger pauses, Variables panel shows component state.

## Troubleshooting Common Issues

## Troubleshooting Common Issues

### Issue 1: ESLint Not Working

**Symptoms**:
- No red squiggles on obvious errors
- Problems panel is empty
- ESLint extension shows "ESLint is disabled"

**Diagnostic Steps**:

1. Check ESLint extension is installed and enabled
2. Open Output panel (`Cmd/Ctrl + Shift + U`)
3. Select "ESLint" from dropdown
4. Look for error messages

**Common Causes**:

**Cause 1: ESLint not installed in project**

```bash
# Solution: Install ESLint
npm install --save-dev eslint
```

**Cause 2: No ESLint config file**

```bash
# Solution: Create .eslintrc.json
npx eslint --init
```

**Cause 3: ESLint disabled in VS Code settings**

```json
// Check .vscode/settings.json - should NOT have:
{
  "eslint.enable": false  // ← Remove this line
}
```

**Cause 4: Wrong working directory**

ESLint looks for config in workspace root. If you opened a parent folder, ESLint won't find the config.

**Solution**: Open the project folder directly in VS Code, not a parent folder.

### Issue 2: Prettier Not Formatting

**Symptoms**:
- Code doesn't format on save
- Manual format command does nothing

**Diagnostic Steps**:

1. Open Command Palette (`Cmd/Ctrl + Shift + P`)
2. Type "Format Document"
3. If prompted, select "Prettier" as formatter

**Common Causes**:

**Cause 1: Prettier not set as default formatter**

```json
// Solution: Add to .vscode/settings.json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

**Cause 2: Format on save not enabled**

```json
// Solution: Add to .vscode/settings.json
{
  "editor.formatOnSave": true
}
```

**Cause 3: Conflicting formatters**

If you have multiple formatter extensions installed (e.g., Beautify, Prettier, ESLint), they may conflict.

**Solution**: Disable or uninstall other formatters, keep only Prettier.

### Issue 3: TypeScript Errors Not Showing

**Symptoms**:
- Type errors in terminal but not in editor
- Red squiggles missing on obvious type errors

**Diagnostic Steps**:

1. Check bottom-right corner of VS Code
2. Look for TypeScript version indicator
3. Click it to see if TypeScript is active

**Common Causes**:

**Cause 1: Using wrong TypeScript version**

```json
// Solution: Add to .vscode/settings.json
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true
}
```

**Cause 2: TypeScript server crashed**

**Solution**: 
1. Open Command Palette
2. Type "TypeScript: Restart TS Server"
3. Press Enter

**Cause 3: File not included in tsconfig.json**

```json
// Check tsconfig.json includes your files
{
  "include": ["**/*.ts", "**/*.tsx"],  // ← Should include your file patterns
  "exclude": ["node_modules"]
}
```

### Issue 4: Debugger Not Hitting Breakpoints

**Symptoms**:
- Breakpoints show gray circle instead of red
- Debugger runs but never pauses
- "Breakpoint ignored because generated code not found" message

**Common Causes**:

**Cause 1: Source maps not enabled**

```json
// Solution: Check next.config.js
module.exports = {
  // Ensure source maps are enabled in development
  productionBrowserSourceMaps: false,  // Keep false for production
  // No need to explicitly enable for dev - enabled by default
};
```

**Cause 2: Wrong URL in launch.json**

```json
// Check .vscode/launch.json
{
  "configurations": [
    {
      "type": "chrome",
      "url": "http://localhost:3000",  // ← Must match your dev server port
      "webRoot": "${workspaceFolder}"
    }
  ]
}
```

**Cause 3: Breakpoint in Server Component**

Server Components run on the server, not in the browser. Use the "debug server-side" configuration instead.

**Solution**: Use the correct debug configuration:
- Client Components: "Next.js: debug client-side"
- Server Components: "Next.js: debug server-side"
- Both: "Next.js: debug full stack"

### Issue 5: Import Auto-Complete Not Working

**Symptoms**:
- Typing component name doesn't suggest import
- Must manually type import statements

**Solution**:

```json
// Add to .vscode/settings.json
{
  "typescript.suggest.autoImports": true,
  "typescript.preferences.includePackageJsonAutoImports": "on",
  "javascript.suggest.autoImports": true
}
```

### Issue 6: Slow Performance

**Symptoms**:
- VS Code feels sluggish
- High CPU usage
- Typing has noticeable lag

**Diagnostic Steps**:

1. Open Command Palette
2. Type "Developer: Show Running Extensions"
3. Look for extensions with high CPU/memory usage

**Common Causes**:

**Cause 1: Too many extensions**

**Solution**: Disable extensions you don't actively use. Keep only essential ones.

**Cause 2: Large node_modules folder**

```json
// Solution: Exclude from file watching
{
  "files.watcherExclude": {
    "**/node_modules/**": true,
    "**/.next/**": true,
    "**/dist/**": true
  }
}
```

**Cause 3: TypeScript checking large files**

```json
// Solution: Exclude generated files from TypeScript
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.disableAutomaticTypeAcquisition": false
}
```

And in `tsconfig.json`:

```json
{
  "exclude": [
    "node_modules",
    ".next",
    "out",
    "dist"
  ]
}
```

## The Journey: From Default to Optimized

## The Journey: From Default to Optimized

### The Setup Evolution Table

| Phase | Configuration | Problems Solved | Time Investment | Productivity Gain |
|-------|--------------|-----------------|-----------------|-------------------|
| 0 | Vanilla VS Code | None | 0 min | Baseline |
| 1 | + ESLint | Catches syntax errors, React mistakes | 10 min | +30% (fewer runtime errors) |
| 2 | + Prettier | Consistent formatting | 5 min | +10% (no formatting debates) |
| 3 | + TypeScript strict | Type safety, refactoring confidence | 15 min | +40% (catch bugs at compile time) |
| 4 | + Debugger | Precise debugging vs. console.log | 10 min | +25% (faster bug diagnosis) |
| 5 | + Custom snippets | Fast boilerplate generation | 20 min | +15% (less typing) |
| **Total** | **Complete setup** | **All of the above** | **60 min** | **~120% productivity increase** |

### Final Configuration Summary

**Extensions Installed** (7 essential):
1. ESLint - Error detection
2. Prettier - Code formatting
3. ES7+ React snippets - Fast boilerplate
4. Error Lens - Inline errors
5. Auto Rename Tag - Paired tag editing
6. Tailwind CSS IntelliSense - Tailwind autocomplete (if using)
7. Import Cost - Bundle size awareness

**Configuration Files Created** (7 files):
1. `.vscode/settings.json` - Editor behavior
2. `.vscode/extensions.json` - Recommended extensions
3. `.vscode/launch.json` - Debugger setup
4. `.eslintrc.json` - Linting rules
5. `.prettierrc` - Formatting rules
6. `tsconfig.json` - TypeScript strict mode
7. `.gitignore` - Version control exclusions

**Custom Snippets Added** (8 snippets):
1. `rfc` - React Function Component
2. `us` - useState hook
3. `ue` - useEffect hook
4. `uq` - React Query useQuery
5. `um` - React Query useMutation
6. `zstore` - Zustand store
7. `nsc` - Next.js Server Component
8. `napi` - Next.js API Route

### Decision Framework: When to Customize Further

**Add more extensions when**:
- You use a specific library frequently (e.g., Prisma extension for Prisma ORM)
- You need specialized tooling (e.g., Docker extension for containerization)
- Team agrees on additional tooling

**Avoid adding extensions when**:
- Functionality is already built into VS Code
- Extension duplicates another extension's features
- Extension significantly impacts performance
- You don't use the feature weekly

**Create custom snippets when**:
- You type the same boilerplate 5+ times per week
- Pattern is team-specific (not covered by existing snippets)
- Snippet saves 30+ seconds per use

**Avoid custom snippets when**:
- Pattern varies too much to template
- Existing snippet is "close enough"
- You'd spend more time maintaining the snippet than typing manually

### Lessons Learned

**1. Configuration is a force multiplier**

One hour of setup saves hours of debugging and formatting debates. The ROI is immediate and compounds over time.

**2. Less is more with extensions**

10 well-chosen extensions beat 50 random ones. Each extension adds cognitive load and potential conflicts.

**3. Team consistency matters more than personal preference**

Shared `.vscode/settings.json` ensures everyone sees the same errors, uses the same formatting, and has the same debugging setup. This eliminates "works on my machine" problems.

**4. The editor should catch errors before the browser**

If you're discovering bugs in the browser that TypeScript or ESLint could have caught, your configuration is too loose. Tighten it.

**5. Debugger > console.log**

Learning the debugger takes 30 minutes. It saves hours over the lifetime of a project. Make the investment.

**6. Snippets are personal productivity tools**

Create snippets for patterns YOU type frequently. Don't cargo-cult someone else's snippet collection.

**7. Configuration evolves with the project**

Start with the essentials. Add tooling as needs emerge. Don't over-configure on day one.

## Quick Reference

## Quick Reference

### Essential Keyboard Shortcuts

**Navigation**:
- `Cmd/Ctrl + P` - Quick Open (jump to file)
- `Cmd/Ctrl + Shift + O` - Go to Symbol (jump to function)
- `F12` - Go to Definition
- `Shift + F12` - Find All References

**Editing**:
- `Cmd/Ctrl + D` - Select Next Occurrence
- `Cmd/Ctrl + /` - Toggle Comment
- `Alt + Up/Down` - Move Line
- `Shift + Alt + Up/Down` - Copy Line

**Code Actions**:
- `Cmd/Ctrl + .` - Quick Fix
- `F2` - Rename Symbol
- `Cmd/Ctrl + Space` - Trigger Autocomplete

**Debugging**:
- `F5` - Start Debugging
- `F9` - Toggle Breakpoint
- `F10` - Step Over
- `F11` - Step Into

### Essential Commands

**Format**:
- `Shift + Alt + F` - Format Document
- `Cmd/Ctrl + K, Cmd/Ctrl + F` - Format Selection

**Search**:
- `Cmd/Ctrl + F` - Find in File
- `Cmd/Ctrl + Shift + F` - Find in Files
- `Cmd/Ctrl + H` - Replace in File

**Terminal**:
- `` Cmd/Ctrl + ` `` - Toggle Terminal
- `Cmd/Ctrl + Shift + ` ` - New Terminal

### Configuration File Locations

```
project-root/
├── .vscode/
│   ├── settings.json       ← Workspace settings
│   ├── extensions.json     ← Recommended extensions
│   └── launch.json         ← Debugger configuration
├── .eslintrc.json          ← ESLint rules
├── .prettierrc             ← Prettier rules
├── tsconfig.json           ← TypeScript configuration
└── .gitignore              ← Git exclusions
```

### Snippet Triggers

| Trigger | Expands To |
|---------|------------|
| `rfc` | React Function Component |
| `us` | useState hook |
| `ue` | useEffect hook |
| `uq` | React Query useQuery |
| `um` | React Query useMutation |
| `zstore` | Zustand store |
| `nsc` | Next.js Server Component |
| `napi` | Next.js API Route |

### Troubleshooting Quick Checks

**ESLint not working?**
1. Check extension installed
2. Check `.eslintrc.json` exists
3. Check Output panel → ESLint

**Prettier not formatting?**
1. Check extension installed
2. Check `editor.formatOnSave: true`
3. Check `editor.defaultFormatter: "esbenp.prettier-vscode"`

**TypeScript errors not showing?**
1. Check bottom-right corner for TS version
2. Restart TS Server (Command Palette)
3. Check `tsconfig.json` includes your files

**Debugger not working?**
1. Check `.vscode/launch.json` exists
2. Check URL matches dev server
3. Check source maps enabled

### Performance Optimization

**If VS Code is slow**:
1. Disable unused extensions
2. Exclude large folders from file watching
3. Exclude generated files from TypeScript checking
4. Close unused editor tabs

### Getting Help

**VS Code Documentation**: https://code.visualstudio.com/docs
**React DevTools**: https://react.dev/learn/react-developer-tools
**TypeScript Handbook**: https://www.typescriptlang.org/docs/handbook/intro.html
**ESLint Rules**: https://eslint.org/docs/rules/
**Prettier Options**: https://prettier.io/docs/en/options.html
