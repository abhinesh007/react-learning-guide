# 03 — React Hooks Deep Dive

> A complete reference and deep dive into every built-in React hook, modern React 19 additions, and authoring production-grade custom hooks.

---

## Table of Contents

- [The Rules of Hooks](#the-rules-of-hooks)
- [State Hooks](#state-hooks)
  - [`useState` Recap & Advanced Patterns](#usestate-recap--advanced-patterns)
  - [`useReducer` for Complex State Logic](#usereducer-for-complex-state-logic)
  - [`useState` vs `useReducer` Decision Guide](#usestate-vs-usereducer-decision-guide)
- [Context Hooks](#context-hooks)
  - [`useContext` & Provider Pattern](#usecontext--provider-pattern)
- [Ref Hooks](#ref-hooks)
  - [`useRef` (DOM Access & Mutable Store)](#useref-dom-access--mutable-store)
  - [`forwardRef` & `useImperativeHandle`](#forwardref--useimperativehandle)
- [Effect Hooks](#effect-hooks)
  - [`useEffect` & `useLayoutEffect` Recap](#useeffect--uselayouteffect-recap)
  - [`useInsertionEffect` (CSS-in-JS)](#useinsertioneffect-css-in-js)
- [Performance & Concurrency Hooks](#performance--concurrency-hooks)
  - [`useMemo`](#usememo)
  - [`useCallback`](#usecallback)
  - [`useTransition`](#usetransition)
  - [`useDeferredValue`](#usedeferredvalue)
- [Utility & Library Hooks](#utility--library-hooks)
  - [`useId` (Accessible IDs)](#useid-accessible-ids)
  - [`useSyncExternalStore` (Concurrent Store Subscriptions)](#usesyncexternalstore-concurrent-store-subscriptions)
  - [`useDebugValue`](#usedebugvalue)
- [React 19 Hooks](#react-19-hooks)
  - [`useActionState`](#useactionstate)
  - [`useFormStatus`](#useformstatus)
  - [`useOptimistic`](#useoptimistic)
  - [`use` API (Promises & Context)](#use-api-promises--context)
- [Custom Hooks](#custom-hooks)
  - [Guidelines & Architecture](#guidelines--architecture)
  - [`useLocalStorage`](#uselocalstorage)
  - [`useDebounce`](#usedebounce)
  - [`useFetch`](#usefetch)
  - [`useMediaQuery`](#usemediaquery)
  - [`useOnClickOutside`](#useonclickoutside)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## The Rules of Hooks

Hooks are regular JavaScript functions that allow functional components to hook into React state and lifecycle features. Because React relies on the **order of execution** to track state between renders, two critical rules apply:

### 1. Only Call Hooks at the Top Level
Never call hooks inside loops, conditions, or nested functions.

```jsx
// ❌ BAD: Hook conditionally called!
if (isLoggedIn) {
  useEffect(() => { ... }, []); // Breaks React's internal call order array!
}

// ✅ GOOD: Always invoke at top level, put conditions INSIDE the hook
useEffect(() => {
  if (isLoggedIn) {
    // Execute logic here
  }
}, [isLoggedIn]);
```

### 2. Only Call Hooks from React Functions
Call them only from React functional components or custom hooks (functions starting with `use`). Do not call them from plain JavaScript helper functions or class components.

### Enforcing Rules via ESLint
Use `eslint-plugin-react-hooks` to automatically catch violations in development:
```json
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

---

## State Hooks

### `useState` Recap & Advanced Patterns

`useState` stores component state across re-renders. Key patterns:
- Initialize lazily with a callback when reading from disk/storage: `useState(() => getInitialVal())`
- Always use the functional updater `setState(prev => ...)` when the new state derives from the old state.

---

### `useReducer` for Complex State Logic

When a component manages complex state transitions, related state fields that change together, or when the next state depends heavily on previous states with multiple action types, `useReducer` is superior to multiple `useState` calls.

```
┌────────────────┐      dispatch(action)     ┌──────────────┐      returns       ┌───────────────┐
│ User / Effect  │ ────────────────────────► │   Reducer    │ ─────────────────► │  Next State   │
└────────────────┘   { type: 'INCREMENT' }   │   Function   │   new state object │ (Re-renders)  │
                                             └──────────────┘                    └───────────────┘
```

#### Complete Shopping Cart Reducer Example

```jsx
import { useReducer } from 'react';

// 1. Initial State
const initialState = {
  items: [],
  discountCode: '',
  discountPercent: 0,
};

// 2. Action Types (Constants or Enums)
const ACTIONS = {
  ADD_ITEM: 'ADD_ITEM',
  REMOVE_ITEM: 'REMOVE_ITEM',
  UPDATE_QUANTITY: 'UPDATE_QUANTITY',
  APPLY_DISCOUNT: 'APPLY_DISCOUNT',
  CLEAR_CART: 'CLEAR_CART',
};

// 3. Pure Reducer Function: (state, action) => newState
function cartReducer(state, action) {
  switch (action.type) {
    case ACTIONS.ADD_ITEM: {
      const existing = state.items.find(item => item.id === action.payload.id);
      if (existing) {
        return {
          ...state,
          items: state.items.map(item =>
            item.id === action.payload.id
              ? { ...item, quantity: item.quantity + 1 }
              : item
          ),
        };
      }
      return {
        ...state,
        items: [...state.items, { ...action.payload, quantity: 1 }],
      };
    }

    case ACTIONS.REMOVE_ITEM:
      return {
        ...state,
        items: state.items.filter(item => item.id !== action.payload.id),
      };

    case ACTIONS.UPDATE_QUANTITY:
      return {
        ...state,
        items: state.items.map(item =>
          item.id === action.payload.id
            ? { ...item, quantity: Math.max(1, action.payload.quantity) }
            : item
        ),
      };

    case ACTIONS.APPLY_DISCOUNT:
      return {
        ...state,
        discountCode: action.payload.code,
        discountPercent: action.payload.percent,
      };

    case ACTIONS.CLEAR_CART:
      return initialState;

    default:
      throw new Error(`Unhandled action type: ${action.type}`);
  }
}

// 4. Component Implementation
export default function ShoppingCart() {
  const [state, dispatch] = useReducer(cartReducer, initialState);

  const totalAmount = state.items.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0
  );
  const discountedTotal = totalAmount * (1 - state.discountPercent / 100);

  return (
    <div className="cart">
      <h2>Shopping Cart ({state.items.length} unique items)</h2>
      <button
        onClick={() =>
          dispatch({
            type: ACTIONS.ADD_ITEM,
            payload: { id: 101, name: 'Wireless Headphones', price: 99 },
          })
        }
      >
        Add Headphones ($99)
      </button>

      <ul>
        {state.items.map(item => (
          <li key={item.id}>
            {item.name} - ${item.price} x {item.quantity}
            <button
              onClick={() =>
                dispatch({
                  type: ACTIONS.UPDATE_QUANTITY,
                  payload: { id: item.id, quantity: item.quantity + 1 },
                })
              }
            >
              +
            </button>
            <button
              onClick={() =>
                dispatch({
                  type: ACTIONS.UPDATE_QUANTITY,
                  payload: { id: item.id, quantity: item.quantity - 1 },
                })
              }
            >
              -
            </button>
            <button
              onClick={() =>
                dispatch({ type: ACTIONS.REMOVE_ITEM, payload: { id: item.id } })
              }
            >
              Remove
            </button>
          </li>
        ))}
      </ul>

      <p>Total: ${discountedTotal.toFixed(2)}</p>
      <button onClick={() => dispatch({ type: ACTIONS.CLEAR_CART })}>
        Empty Cart
      </button>
    </div>
  );
}
```

---

### `useState` vs `useReducer` Decision Guide

| Criterion | `useState` | `useReducer` |
|:---|:---|:---|
| **Complexity** | Simple primitives, independent flags | Complex objects, dependent fields |
| **Logic Location** | Mixed directly inside event handlers | Centralized, pure function testable in isolation |
| **State Transitions** | Direct overwrites (`setIsOpen(true)`) | Predictable action events (`dispatch({ type })`) |
| **Debugging** | Harder to trace series of updates | Easily traceable via Redux DevTools or logger |
| **Testing** | Requires rendering the component | Test the pure reducer function directly with unit tests |

---

## Context Hooks

### `useContext` & Provider Pattern

The Context API enables passing data through the component tree without manually threading props down at every level (eliminates "prop drilling").

```jsx
import { createContext, useContext, useState } from 'react';

// 1. Create Context with default value
const ThemeContext = createContext({
  theme: 'light',
  toggleTheme: () => {},
});

// 2. Custom Provider Component
export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');

  const toggleTheme = () => {
    setTheme(prev => (prev === 'light' ? 'dark' : 'light'));
  };

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      <div className={`app-wrapper ${theme}`}>{children}</div>
    </ThemeContext.Provider>
  );
}

// 3. Custom Consumer Hook (Clean abstraction)
export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within a ThemeProvider');
  }
  return context;
}

// 4. Nested Child Consuming Context
export function HeaderNav() {
  const { theme, toggleTheme } = useTheme();

  return (
    <nav>
      <span>Current mode: {theme}</span>
      <button onClick={toggleTheme}>Toggle Theme</button>
    </nav>
  );
}
```

---

## Ref Hooks

### `useRef` (DOM Access & Mutable Store)

`useRef` returns a mutable object with a `.current` property:
1. **DOM References:** Pointing directly to a real HTML element.
2. **Mutable Instance Variables:** Storing values across renders **without** causing a re-render when modified.

```jsx
import { useRef, useEffect, useState } from 'react';

export function SearchWithRef() {
  // Use Case 1: Direct DOM element access
  const inputRef = useRef(null);

  // Use Case 2: Persisting values across renders without re-rendering
  const renderCount = useRef(0);
  const timerRef = useRef(null);

  const [query, setQuery] = useState('');

  // Increment render counter on every render without triggering a loop!
  renderCount.current += 1;

  useEffect(() => {
    // Automatically focus input on mount
    inputRef.current?.focus();
  }, []);

  const handleStartTimer = () => {
    if (timerRef.current) return;
    timerRef.current = setInterval(() => {
      console.log('Tick at', new Date().toLocaleTimeString());
    }, 1000);
  };

  const handleStopTimer = () => {
    clearInterval(timerRef.current);
    timerRef.current = null;
  };

  return (
    <div>
      <p>Component has rendered: {renderCount.current} times</p>
      <input
        ref={inputRef}
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Type to search..."
      />
      <button onClick={handleStartTimer}>Start Timer</button>
      <button onClick={handleStopTimer}>Stop Timer</button>
    </div>
  );
}
```

---

### `forwardRef` & `useImperativeHandle`

Normally, parent components cannot pass `ref` to child function components. `forwardRef` passes the ref down, while `useImperativeHandle` allows the child to expose a curated, custom public API instead of the raw DOM node.

> 💡 *Note for React 19:* In React 19, `ref` can be received as a regular prop in function components, rendering `forwardRef` optional! However, `forwardRef` remains standard in React 18 codebases.

```jsx
import { forwardRef, useRef, useImperativeHandle } from 'react';

// Child component exposing imperative actions
const CustomTextInput = forwardRef((props, ref) => {
  const innerInputRef = useRef(null);

  // Expose ONLY specific methods to the parent
  useImperativeHandle(ref, () => ({
    focusAndHighlight: () => {
      innerInputRef.current?.focus();
      innerInputRef.current.style.borderColor = '#3b82f6';
    },
    resetInput: () => {
      if (innerInputRef.current) {
        innerInputRef.current.value = '';
      }
    },
  }));

  return <input ref={innerInputRef} {...props} />;
});

// Parent invoking child's imperative methods
export function FormManager() {
  const customRef = useRef(null);

  return (
    <div>
      <CustomTextInput ref={customRef} placeholder="Enter your name" />
      <button onClick={() => customRef.current?.focusAndHighlight()}>
        Focus Child
      </button>
      <button onClick={() => customRef.current?.resetInput()}>
        Clear Child
      </button>
    </div>
  );
}
```

---

## Effect Hooks

### `useEffect` & `useLayoutEffect` Recap

- **`useEffect`**: Asynchronous, runs **after** the screen has been painted. Ideal for data fetching, subscriptions, and logging.
- **`useLayoutEffect`**: Synchronous, runs **before** the browser paints the screen. Use exclusively when measuring DOM dimensions or calculating layout to prevent visual flicker.

### `useInsertionEffect` (CSS-in-JS)

Introduced in React 18, `useInsertionEffect` fires **synchronously before DOM mutations**. It is intended strictly for authoring CSS-in-JS libraries (like Emotion or Styled Components) to inject `<style>` tags before React calculates the layout, avoiding layout thrashing.

```jsx
import { useInsertionEffect } from 'react';

function useDynamicStyleRule(rule) {
  useInsertionEffect(() => {
    const styleTag = document.createElement('style');
    styleTag.textContent = rule;
    document.head.appendChild(styleTag);

    return () => {
      document.head.removeChild(styleTag);
    };
  }, [rule]);
}
```

---

## Performance & Concurrency Hooks

### `useMemo`

`useMemo` caches the result of an expensive calculation between re-renders until one of its dependencies changes.

```jsx
import { useState, useMemo } from 'react';

function ProductCatalog({ products }) {
  const [filterQuery, setFilterQuery] = useState('');
  const [sortBy, setSortBy] = useState('price');

  // Expensive calculation cached unless products, filterQuery, or sortBy changes
  const processedProducts = useMemo(() => {
    console.log('Calculating filtered products...');
    return products
      .filter(p => p.name.toLowerCase().includes(filterQuery.toLowerCase()))
      .sort((a, b) => (sortBy === 'price' ? a.price - b.price : a.name.localeCompare(b.name)));
  }, [products, filterQuery, sortBy]);

  return (
    <div>
      <input value={filterQuery} onChange={e => setFilterQuery(e.target.value)} />
      <p>Results count: {processedProducts.length}</p>
    </div>
  );
}
```

---

### `useCallback`

In JavaScript, functions declared inside a component are recreated with a new memory reference on every render. If passed to memoized child components (`React.memo`), those children re-render unnecessarily. `useCallback` preserves the same function instance between renders.

```jsx
import { useState, useCallback, memo } from 'react';

// Child component memoized with React.memo
const FilterButton = memo(function FilterButton({ onClick, label }) {
  console.log(`Rendered button: ${label}`);
  return <button onClick={onClick}>{label}</button>;
});

export function ParentDashboard() {
  const [count, setCount] = useState(0);

  // Without useCallback, this function is a new reference every time count updates!
  const handleApplyFilter = useCallback(() => {
    console.log('Filter applied');
  }, []); // Empty dependencies: same instance forever

  return (
    <div>
      <p>Parent counter: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment Parent</button>

      {/* FilterButton will NOT re-render when count updates */}
      <FilterButton onClick={handleApplyFilter} label="Apply Global Filter" />
    </div>
  );
}
```

---

### `useTransition`

`useTransition` lets you mark state updates as non-urgent transitions, keeping the user interface responsive during heavy UI updates (like rendering 10,000 search results).

```jsx
import { useState, useTransition } from 'react';

export function TabNavigator() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState('home');

  const selectTab = (nextTab) => {
    // Non-urgent update: will not freeze clicks or typing!
    startTransition(() => {
      setTab(nextTab);
    });
  };

  return (
    <div>
      <button onClick={() => selectTab('home')}>Home</button>
      <button onClick={() => selectTab('heavy-reports')}>Heavy Reports</button>
      
      {isPending && <span className="spinner"> Loading tab...</span>}
      
      <main>
        {tab === 'home' && <p>Welcome to Home</p>}
        {tab === 'heavy-reports' && <HeavyReportsView />}
      </main>
    </div>
  );
}

function HeavyReportsView() {
  const items = Array.from({ length: 5000 }, (_, i) => `Report #${i + 1}`);
  return (
    <ul>
      {items.map(item => (
        <li key={item}>{item}</li>
      ))}
    </ul>
  );
}
```

---

### `useDeferredValue`

`useDeferredValue` accepts a value and returns a deferred copy that lags behind during urgent updates. It provides a similar benefit to debouncing/throttling without arbitrary time delays.

```jsx
import { useState, useDeferredValue, memo } from 'react';

const HeavyList = memo(function HeavyList({ query }) {
  const items = Array.from({ length: 4000 }, (_, i) => `Result matching "${query}" - item #${i}`);
  return (
    <ul>
      {items.map(item => (
        <li key={item}>{item}</li>
      ))}
    </ul>
  );
});

export function SearchContainer() {
  const [text, setText] = useState('');
  // Urgent update: text changes instantly for smooth keystrokes
  // Deferred update: deferredText lags until the main thread has time to re-render
  const deferredText = useDeferredValue(text);

  return (
    <div>
      <input value={text} onChange={e => setText(e.target.value)} placeholder="Type quickly..." />
      <div style={{ opacity: text !== deferredText ? 0.6 : 1 }}>
        <HeavyList query={deferredText} />
      </div>
    </div>
  );
}
```

---

## Utility & Library Hooks

### `useId` (Accessible IDs)

Generates unique, stable identifiers across client and server renders (SSR-safe). Perfect for linking HTML labels to inputs and ARIA attributes:

```jsx
import { useId } from 'react';

export function FormField({ label, type = 'text' }) {
  const id = useId();
  const hintId = `${id}-hint`;

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} type={type} aria-describedby={hintId} />
      <span id={hintId}>Please provide a valid value</span>
    </div>
  );
}
```

---

### `useSyncExternalStore` (Concurrent Store Subscriptions)

Designed for library authors to subscribe safely to external stores outside of React (such as browser window size, redux store, navigator online status) without tearing under concurrent rendering.

```jsx
import { useSyncExternalStore } from 'react';

// External API subscription helpers
function subscribeOnlineStatus(callback) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function getOnlineSnapshot() {
  return navigator.onLine;
}

export function useOnlineStatus() {
  return useSyncExternalStore(subscribeOnlineStatus, getOnlineSnapshot);
}

// Consuming component:
export function ConnectionBanner() {
  const isOnline = useOnlineStatus();
  return (
    <div style={{ background: isOnline ? '#10b981' : '#ef4444', color: '#fff', padding: 8 }}>
      Status: {isOnline ? 'Online' : 'Offline — Check your connection'}
    </div>
  );
}
```

---

### `useDebugValue`

Enables formatting and custom labeling inside React Developer Tools for your custom hooks.

```jsx
import { useState, useDebugValue } from 'react';

export function useFriendStatus(friendId) {
  const [isOnline, setIsOnline] = useState(null);

  // Shows: FriendStatus: "Online" in React DevTools Inspector
  useDebugValue(isOnline ? 'Online' : 'Offline');

  return isOnline;
}
```

---

## React 19 Hooks

React 19 introduces powerful hooks focused on actions, form handling, and asynchronous data reading.

### `useActionState`

Manages asynchronous form actions, providing current state, an action trigger, and a pending boolean flag.

```jsx
import { useActionState } from 'react';

async function updateUsernameAction(previousState, formData) {
  const newName = formData.get('username');
  try {
    const res = await fetch('/api/user/rename', {
      method: 'POST',
      body: JSON.stringify({ name: newName }),
    });
    if (!res.ok) throw new Error('Failed to update name');
    return { name: newName, error: null };
  } catch (err) {
    return { ...previousState, error: err.message };
  }
}

export function UserRenameForm() {
  const [state, formAction, isPending] = useActionState(updateUsernameAction, {
    name: 'Abhi',
    error: null,
  });

  return (
    <form action={formAction}>
      <input name="username" defaultValue={state.name} />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Save Name'}
      </button>
      {state.error && <p style={{ color: 'red' }}>{state.error}</p>}
    </form>
  );
}
```

---

### `useFormStatus`

Returns the status of the parent `<form>` without prop drilling. Must be called from a component rendered *inside* the `<form>`.

```jsx
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending, data, method } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit Post'}
    </button>
  );
}
```

---

### `useOptimistic`

Enables optimistic UI updates, immediately reflecting intended changes on screen while background network operations execute.

```jsx
import { useOptimistic, useState } from 'react';

export function LikeWidget({ initialLikes }) {
  const [likes, setLikes] = useState(initialLikes);

  // Optimistic state wrapper
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    likes,
    (current, updateAmount) => current + updateAmount
  );

  const handleLike = async () => {
    // 1. Immediately increment on screen
    addOptimisticLike(1);

    // 2. Perform background network sync
    const res = await fetch('/api/like', { method: 'POST' });
    const data = await res.json();
    setLikes(data.likes);
  };

  return (
    <button onClick={handleLike}>
      ❤️ Likes: {optimisticLikes}
    </button>
  );
}
```

---

### `use` API (Promises & Context)

The `use` API is a unified primitive that allows reading resources (Promises or Contexts) conditionally and within render loops:

```jsx
import { use, Suspense } from 'react';

function UserProfile({ userPromise }) {
  // Directly unwraps the Promise! If pending, suspends the parent Suspense boundary
  const user = use(userPromise);

  return <div>Welcome, {user.name}!</div>;
}

export function App() {
  const userPromise = fetch('/api/me').then(r => r.json());

  return (
    <Suspense fallback={<div>Loading user...</div>}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}
```

---

## Custom Hooks

Custom hooks are JavaScript functions whose names begin with `use` and can call other React hooks. They are the primary abstraction for extracting and reusing stateful logic across components.

### Guidelines & Architecture
1. **Always prefix with `use`** (enforces ESLint linter checks).
2. **Pure business logic:** Keep presentation JSX out of custom hooks.
3. **Return ergonomic structures:** Return an object `{ data, error, isLoading }` or tuple `[value, setValue]`.

---

### `useLocalStorage`

Persists state to the browser's `localStorage` with JSON serialization and cross-tab synchronization.

```jsx
import { useState, useEffect } from 'react';

export function useLocalStorage(key, initialValue) {
  // Read once from storage lazily
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.warn(`Error reading localStorage key "${key}":`, error);
      return initialValue;
    }
  });

  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(storedValue));
    } catch (error) {
      console.warn(`Error writing to localStorage key "${key}":`, error);
    }
  }, [key, storedValue]);

  return [storedValue, setStoredValue];
}
```

---

### `useDebounce`

Delays updating a value until a specified delay has elapsed since the last change.

```jsx
import { useState, useEffect } from 'react';

export function useDebounce(value, delayMs = 300) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delayMs);

    return () => {
      clearTimeout(handler);
    };
  }, [value, delayMs]);

  return debouncedValue;
}
```

---

### `useFetch`

Robust generic fetch hook with loading, error, and cancellation support.

```jsx
import { useState, useEffect } from 'react';

export function useFetch(url, options = {}) {
  const [data, setData] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    setIsLoading(true);
    setError(null);

    fetch(url, { ...options, signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP Error: ${res.status}`);
        return res.json();
      })
      .then(result => {
        setData(result);
        setIsLoading(false);
      })
      .catch(err => {
        if (err.name !== 'AbortError') {
          setError(err.message);
          setIsLoading(false);
        }
      });

    return () => controller.abort();
  }, [url]);

  return { data, isLoading, error };
}
```

---

### `useMediaQuery`

Subscribes to CSS media query matching state.

```jsx
import { useState, useEffect } from 'react';

export function useMediaQuery(query) {
  const [matches, setMatches] = useState(() => {
    if (typeof window !== 'undefined') {
      return window.matchMedia(query).matches;
    }
    return false;
  });

  useEffect(() => {
    const mediaQueryList = window.matchMedia(query);
    const listener = (event) => setMatches(event.matches);

    mediaQueryList.addEventListener('change', listener);
    setMatches(mediaQueryList.matches);

    return () => mediaQueryList.removeEventListener('change', listener);
  }, [query]);

  return matches;
}
```

---

### `useOnClickOutside`

Detects clicks outside of a specified element (for dropdowns, modals, and tooltips).

```jsx
import { useEffect } from 'react';

export function useOnClickOutside(ref, handler) {
  useEffect(() => {
    const listener = (event) => {
      // Do nothing if clicking ref's element or its children
      if (!ref.current || ref.current.contains(event.target)) {
        return;
      }
      handler(event);
    };

    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);

    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]);
}
```

---

## 🏋️ Practice Exercises

### Exercise 1: Build `useToggle`
Create a custom hook `useToggle(initialValue = false)` that returns:
- The boolean value
- A function to toggle the value
- Optional helpers to explicitly set `true` or `false`

### Exercise 2: Build `usePrevious`
Create a custom hook `usePrevious(value)` that returns the value from the **previous render** using `useRef` and `useEffect`.

### Exercise 3: Build `useWindowSize`
Create a custom hook returning `{ width, height }` that throttles/debounces window resize events and provides clean teardown.

---

> **Previous:** [← 02 — State & Lifecycle](./02-state-and-lifecycle.md)  
> **Next:** [04 — Advanced Patterns →](./04-advanced-patterns.md)
