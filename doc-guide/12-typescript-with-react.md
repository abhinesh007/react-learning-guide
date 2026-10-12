# 12 — TypeScript with React

> Complete guide to building enterprise-grade, fully type-safe React applications: typing props, hooks, synthetic events, generics, discriminated unions, and polymorphic components.

---

## Table of Contents

- [Why TypeScript with React?](#why-typescript-with-react)
- [Project Setup with Vite](#project-setup-with-vite)
- [Typing Components & Props](#typing-components--props)
  - [`interface` vs. `type` for Props](#interface-vs-type-for-props)
  - [Optional Props & Default Values](#optional-props--default-values)
  - [Typing `children`: `React.ReactNode` vs. `PropsWithChildren`](#typing-children-reactreactnode-vs-propswithchildren)
- [Typing React Hooks](#typing-react-hooks)
  - [`useState<T>` Inference & Explicit Types](#usestatet-inference--explicit-types)
  - [`useRef` (DOM vs. Mutable Values)](#useref-dom-vs-mutable-values)
  - [`useReducer` with Discriminated Unions](#usereducer-with-discriminated-unions)
  - [`useContext` with Safe Non-Null Patterns](#usecontext-with-safe-non-null-patterns)
- [Typing Synthetic Events](#typing-synthetic-events)
  - [Event Reference Table](#event-reference-table)
  - [Typing Inline vs. Standalone Event Handlers](#typing-inline-vs-standalone-event-handlers)
- [Typing Styles: `React.CSSProperties`](#typing-styles-reactcssproperties)
- [Generic React Components](#generic-react-components)
  - [Building a Type-Safe Reusable `<List<T>>`](#building-a-type-safe-reusable-listt)
  - [Building a Type-Safe Dropdown `<Select<T>>`](#building-a-type-safe-dropdown-selectt)
- [React Utility Types](#react-utility-types)
  - [`ComponentProps` & `ComponentPropsWithoutRef`](#componentprops--componentpropswithoutref)
  - [`HTMLAttributes`](#htmlattributes)
  - [TypeScript Utilities: `Omit`, `Pick`, `Extract`](#typescript-utilities-omit-pick-extract)
- [Advanced TypeScript Patterns in React](#advanced-typescript-patterns-in-react)
  - [Discriminated Unions for UI Variants](#discriminated-unions-for-ui-variants)
  - [Exhaustive Checking with `never`](#exhaustive-checking-with-never)
  - [Polymorphic `as` Prop Component](#polymorphic-as-prop-component)
- [Type-Safe API Responses](#type-safe-api-responses)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## Why TypeScript with React?

1. **Self-Documenting Codebases:** Props, callback signatures, and event objects have autocomplete in VS Code without needing to open source files.
2. **Instant Refactoring:** Renaming a prop or changing a data shape immediately flags all affected files at compile time.
3. **Eliminates Null & Undefined Runtime Errors:** Catches `cannot read property of undefined` before code ever hits production.

---

## Project Setup with Vite

```bash
# Initialize a new React + TypeScript template
npm create vite@latest my-ts-react-app -- --template react-ts
cd my-ts-react-app
npm install
```

---

## Typing Components & Props

### `interface` vs. `type` for Props

Both `interface` and `type` can type props. In React development:
- Use `interface` for public component prop definitions (extendable).
- Use `type` for unions, intersections, or primitives.

```tsx
// src/components/UserProfile.tsx
interface UserProfileProps {
  id: string;
  username: string;
  age?: number; // Optional prop
  role: 'admin' | 'editor' | 'viewer'; // String literal union
  onUpdateRole: (newRole: 'admin' | 'editor' | 'viewer') => void;
}

export function UserProfile({
  username,
  age = 18, // Default value
  role,
  onUpdateRole,
}: UserProfileProps) {
  return (
    <div>
      <h2>{username} ({age})</h2>
      <p>Role: {role}</p>
      <button onClick={() => onUpdateRole('admin')}>Promote</button>
    </div>
  );
}
```

---

### Typing `children`: `React.ReactNode` vs. `PropsWithChildren`

`React.ReactNode` represents anything that can be rendered inside JSX (strings, numbers, JSX elements, fragments, booleans, arrays, null).

```tsx
import React, { ReactNode } from 'react';

// Approach A: Explicit ReactNode
interface CardProps {
  title: string;
  children: ReactNode;
}

export function Card({ title, children }: CardProps) {
  return (
    <div className="card">
      <h3>{title}</h3>
      <div>{children}</div>
    </div>
  );
}

// Approach B: React.PropsWithChildren helper
type SectionProps = React.PropsWithChildren<{
  headerText: string;
}>;

export function Section({ headerText, children }: SectionProps) {
  return (
    <section>
      <h2>{headerText}</h2>
      {children}
    </section>
  );
}
```

---

## Typing React Hooks

### `useState<T>` Inference & Explicit Types

TypeScript automatically infers primitive types. Explicit generic typing is required when initial values are `null` or complex unions:

```tsx
import { useState } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
}

export function MemberView() {
  // 1. Primitive inferred automatically as boolean
  const [isOpen, setIsOpen] = useState(false);

  // 2. Explicit Union: User or null
  const [user, setUser] = useState<User | null>(null);

  // 3. Array of typed items
  const [tags, setTags] = useState<string[]>([]);

  return <div>{user ? user.name : 'No user'}</div>;
}
```

---

### `useRef` (DOM vs. Mutable Values)

`useRef` has two distinct typing scenarios:

#### 1. DOM Element References (Read-only `.current`)
Pass the exact HTML element type and initialize with `null`:

```tsx
import { useRef, useEffect } from 'react';

export function AutoFocusInput() {
  // Element is initially null before mount, then HTMLInputElement
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    // Optional chaining required because inputRef.current might be null
    inputRef.current?.focus();
  }, []);

  return <input ref={inputRef} type="text" />;
}
```

#### 2. Mutable Instance Variable (Writable `.current`)
Do not pass `null` if you want a mutable instance variable:

```tsx
const timerId = useRef<number | undefined>(undefined);
timerId.current = window.setInterval(() => {}, 1000); // Fully writable!
```

---

### `useReducer` with Discriminated Unions

Discriminated unions ensure you cannot dispatch an action with the wrong payload shape:

```tsx
import { useReducer } from 'react';

interface State {
  count: number;
  step: number;
}

// Discriminated union of actions using `type` discriminator
type Action =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'SET_STEP'; payload: number }
  | { type: 'RESET'; initialCount?: number };

function counterReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + state.step };
    case 'DECREMENT':
      return { ...state, count: state.count - state.step };
    case 'SET_STEP':
      // TypeScript guarantees action.payload exists and is a number!
      return { ...state, step: action.payload };
    case 'RESET':
      return { ...state, count: action.initialCount ?? 0 };
    default:
      return state;
  }
}
```

---

### `useContext` with Safe Non-Null Patterns

Avoid checking `if (!context)` in every single component by writing a typed custom hook:

```tsx
import { createContext, useContext, ReactNode, useState } from 'react';

interface AuthContextType {
  token: string | null;
  setToken: (token: string | null) => void;
  logout: () => void;
}

// Initialize with null
const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [token, setToken] = useState<string | null>(null);

  const logout = () => setToken(null);

  return (
    <AuthContext.Provider value={{ token, setToken, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// Custom hook guarantees non-null context or throws helpful error
export function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be consumed within an <AuthProvider>');
  }
  return context;
}
```

---

## Typing Synthetic Events

React wraps native browser events in its own `React.SyntheticEvent` wrapper.

### Event Reference Table

| Event | Type Signature | Common Target Elements |
|:---|:---|:---|
| Text input change | `React.ChangeEvent<HTMLInputElement>` | `<input type="text">` |
| Textarea change | `React.ChangeEvent<HTMLTextAreaElement>` | `<textarea>` |
| Select change | `React.ChangeEvent<HTMLSelectElement>` | `<select>` |
| Form submission | `React.FormEvent<HTMLFormElement>` | `<form>` |
| Button click | `React.MouseEvent<HTMLButtonElement>` | `<button>` |
| Div click | `React.MouseEvent<HTMLDivElement>` | `<div>` |
| Keyboard press | `React.KeyboardEvent<HTMLInputElement>` | `<input>` |
| Focus / Blur | `React.FocusEvent<HTMLInputElement>` | `<input>` |

---

### Typing Inline vs. Standalone Event Handlers

```tsx
export function EventExamples() {
  // 1. Standalone handler: Must explicitly type the event parameter
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    console.log(e.target.value);
  };

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" onChange={handleChange} />

      {/* 2. Inline handler: TypeScript infers 'e' automatically! */}
      <button onClick={(e) => console.log(e.clientX)}>Submit</button>
    </form>
  );
}
```

---

## Typing Styles: `React.CSSProperties`

```tsx
import React from 'react';

interface BannerProps {
  customStyle?: React.CSSProperties;
}

export function Banner({ customStyle }: BannerProps) {
  const baseStyle: React.CSSProperties = {
    padding: '16px',
    display: 'flex',
    backgroundColor: '#3b82f6',
    ...customStyle,
  };

  return <div style={baseStyle}>Banner Content</div>;
}
```

---

## Generic React Components

### Building a Type-Safe Reusable `<List<T>>`

Generic components allow consumers to render lists of arbitrary data structures while retaining 100% type safety:

```tsx
import React from 'react';

interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyExtractor: (item: T) => string | number;
  emptyMessage?: string;
}

export function List<T>({
  items,
  renderItem,
  keyExtractor,
  emptyMessage = 'No items found',
}: ListProps<T>) {
  if (items.length === 0) {
    return <p>{emptyMessage}</p>;
  }

  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>
          {renderItem(item, index)}
        </li>
      ))}
    </ul>
  );
}
```

#### Consumer Usage (Fully Type-Checked):
```tsx
interface Book {
  id: number;
  title: string;
  author: string;
}

const books: Book[] = [
  { id: 1, title: 'Clean Code', author: 'Robert C. Martin' },
  { id: 2, title: 'Pragmatic Programmer', author: 'Andy Hunt' },
];

export function BookCatalog() {
  return (
    <List
      items={books}
      keyExtractor={(book) => book.id}
      renderItem={(book) => (
        <span>{book.title} by {book.author}</span> // book is strictly typed as Book!
      )}
    />
  );
}
```

---

## React Utility Types

### `ComponentProps` & `ComponentPropsWithoutRef`

Extract all native HTML attributes from a tag without typing them manually:

```tsx
import React, { ComponentPropsWithoutRef } from 'react';

// Extends all native HTML <button> props (onClick, disabled, type, etc.)
interface ButtonProps extends ComponentPropsWithoutRef<'button'> {
  variant?: 'primary' | 'secondary' | 'danger';
}

export function Button({ variant = 'primary', className, ...props }: ButtonProps) {
  return (
    <button className={`btn btn-${variant} ${className ?? ''}`} {...props} />
  );
}

// Usage: native attributes like 'type="submit"' or 'disabled' work automatically!
// <Button type="submit" disabled variant="danger">Delete</Button>
```

---

## Advanced TypeScript Patterns in React

### Discriminated Unions for UI Variants

Prevent impossible states. A component should either be in a loading state OR have data, never both:

```tsx
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

export function DisplayUser({ state }: { state: AsyncState<{ name: string }> }) {
  switch (state.status) {
    case 'loading':
      return <div>Loading...</div>;
    case 'error':
      return <div>Error: {state.error.message}</div>;
    case 'success':
      // TypeScript guarantees state.data exists here!
      return <div>Welcome, {state.data.name}!</div>;
    case 'idle':
      return <div>Ready to search</div>;
  }
}
```

---

### Exhaustive Checking with `never`

Ensure all cases of a union are handled at compile time:

```tsx
function assertNever(x: never): never {
  throw new Error(`Unhandled union value: ${JSON.stringify(x)}`);
}

type NotificationType = 'info' | 'warning' | 'error';

function getIcon(type: NotificationType) {
  switch (type) {
    case 'info': return 'ℹ️';
    case 'warning': return '⚠️';
    case 'error': return '❌';
    default:
      // If a new type is added (e.g. 'success') and not handled, TS errors here!
      return assertNever(type);
  }
}
```

---

### Polymorphic `as` Prop Component

Allows a component to dynamically render as a `<div>`, `<button>`, or `<a>` with full type safety for that tag's attributes:

```tsx
import React from 'react';

type TextProps<C extends React.ElementType> = {
  as?: C;
  children: React.ReactNode;
} & React.ComponentPropsWithoutRef<C>;

export function Text<C extends React.ElementType = 'span'>({
  as,
  children,
  ...rest
}: TextProps<C>) {
  const Component = as || 'span';
  return <Component {...rest}>{children}</Component>;
}

// Usage:
// <Text as="h1">Header</Text>
// <Text as="a" href="https://google.com" target="_blank">Link</Text>
```

---

## Type-Safe API Responses

```tsx
interface ApiResponse<T> {
  data: T;
  meta: { total: number; page: number };
}

interface Product {
  id: string;
  price: number;
}

async function getProducts(): Promise<ApiResponse<Product[]>> {
  const res = await fetch('/api/products');
  if (!res.ok) throw new Error('API failed');
  return res.json() as Promise<ApiResponse<Product[]>>;
}
```

---

## 🏋️ Practice Exercises

### Exercise 1: Convert a Component to Strict TypeScript
Take a plain JavaScript component that accepts `items`, `onSelect`, `filterKey`, and `selectedId`, and convert it to a generic typed component with no `any` types.

### Exercise 2: Discriminated Union Modal
Create a `<Modal>` component where:
- If `variant="confirm"`, it requires `onConfirm: () => void`.
- If `variant="alert"`, `onConfirm` is forbidden, requiring only `onAcknowledge: () => void`.

---

> **Previous:** [← 11 — Performance Optimization](./11-performance-optimization.md)  
> **Next:** [13 — Real-World Patterns →](./13-real-world-patterns.md)
