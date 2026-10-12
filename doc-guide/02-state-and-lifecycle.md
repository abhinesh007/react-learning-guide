# 02 — State & Lifecycle

> Master React's dynamic nature — understand how data changes trigger UI updates, how the component lifecycle functions, and how to harness `useState` and `useEffect` without footguns.

---

## Table of Contents

- [What is State?](#what-is-state)
- [State vs Props](#state-vs-props)
- [The `useState` Hook](#the-usestate-hook)
  - [Basic Usage](#basic-usage)
  - [State with Different Primitive Types](#state-with-different-primitive-types)
  - [The Functional Updater Pattern](#the-functional-updater-pattern)
  - [Lazy Initial State](#lazy-initial-state)
- [Updating Objects in State](#updating-objects-in-state)
  - [Immutability Principle](#immutability-principle)
  - [Shallow Object Updates](#shallow-object-updates)
  - [Nested Object Updates](#nested-object-updates)
  - [Deep Clones & Helpers](#deep-clones--helpers)
- [Updating Arrays in State](#updating-arrays-in-state)
  - [Adding Items](#adding-items)
  - [Removing Items](#removing-items)
  - [Updating Items](#updating-items)
  - [Sorting and Reordering](#sorting-and-reordering)
- [Component Lifecycle (Conceptual)](#component-lifecycle-conceptual)
  - [Mounting, Updating, Unmounting](#mounting-updating-unmounting)
  - [Class Lifecycle vs. Hook Model](#class-lifecycle-vs-hook-model)
- [The `useEffect` Hook](#the-useeffect-hook)
  - [Mental Model: Synchronization](#mental-model-synchronization)
  - [The Dependency Array Demystified](#the-dependency-array-demystified)
  - [The Cleanup Function](#the-cleanup-function)
  - [Common Use Cases](#common-use-cases)
- [Common `useEffect` Pitfalls & Traps](#common-useeffect-pitfalls--traps)
  - [Infinite Render Loops](#infinite-render-loops)
  - [Stale Closures & Missing Dependencies](#stale-closures--missing-dependencies)
  - [Data Fetching Race Conditions](#data-fetching-race-conditions)
  - [Object and Array Dependencies](#object-and-array-dependencies)
- [`useLayoutEffect` vs `useEffect`](#uselayouteffect-vs-useeffect)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## What is State?

In React, **State** is a component's private, mutable memory. While **props** pass data down from parent to child, **state** holds data that can change over time based on user interactions, network responses, timers, or internal system events.

Whenever state updates:
1. React schedules a re-render of that component.
2. The component function runs again from top to bottom with the new state values.
3. React compares the newly returned JSX Virtual DOM tree with the previous one.
4. Only the real DOM nodes that actually changed get committed to the browser screen.

```
┌─────────────────┐       User clicks / API fires
│  Initial State  │ ──────────────────────────────────┐
└────────┬────────┘                                   ▼
         │                                   ┌────────────────┐
         ▼                                   │ `setState(...)`│
┌─────────────────┐                          └────────┬───────┘
│ Initial Render  │                                   │
└────────┬────────┘                          Schedules Re-render
         │                                            ▼
         ▼                                   ┌────────────────┐
┌─────────────────┐                          │  New Render &  │
│  Mounted in DOM │ ◄────────────────────────┤  Virtual Diff  │
└─────────────────┘      Committed to DOM    └────────────────┘
```

---

## State vs Props

| Dimension | Props | State |
|:---|:---|:---|
| **Origin** | Passed from parent component | Initialized within component |
| **Mutability** | Read-only (immutable to the child) | Mutable only via setter function (`setState`) |
| **Ownership** | Owned by the parent | Owned by the component itself |
| **Triggers re-render?** | Yes, if new prop values differ | Yes, when updated with a new value |
| **Analogy** | Function arguments | Function local variables (that persist across calls) |

---

## The `useState` Hook

`useState` is the foundational React hook for declaring state variables in functional components.

### Basic Usage

```jsx
import { useState } from 'react';

function Counter() {
  // Array destructuring: [currentValue, setterFunction]
  const [count, setCount] = useState(0);

  return (
    <div className="card">
      <h2>Count: {count}</h2>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(count - 1)}>Decrement</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}

export default Counter;
```

---

### State with Different Primitive Types

```jsx
import { useState } from 'react';

function UserPreferences() {
  // String
  const [theme, setTheme] = useState('light');
  
  // Number
  const [fontSize, setFontSize] = useState(16);
  
  // Boolean
  const [notificationsEnabled, setNotificationsEnabled] = useState(true);

  return (
    <div>
      <p>Theme: {theme}</p>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>

      <p>Font Size: {fontSize}px</p>
      <button onClick={() => setFontSize(prev => prev + 2)}>Increase Font</button>

      <label>
        <input 
          type="checkbox" 
          checked={notificationsEnabled} 
          onChange={(e) => setNotificationsEnabled(e.target.checked)} 
        />
        Enable Notifications
      </label>
    </div>
  );
}
```

---

### The Functional Updater Pattern

When the new state depends on the **previous state**, always pass an updater function: `setCount(prev => prev + 1)`.

#### ❌ The Bug: Direct state reference in consecutive calls

```jsx
function BrokenCounter() {
  const [count, setCount] = useState(0);

  const handleTripleIncrement = () => {
    // ❌ count is captured as 0 in this render scope!
    setCount(count + 1); // setCount(0 + 1)
    setCount(count + 1); // setCount(0 + 1)
    setCount(count + 1); // setCount(0 + 1)
    // Result: count becomes 1, NOT 3!
  };

  return <button onClick={handleTripleIncrement}>Add 3 (Broken)</button>;
}
```

#### ✅ The Fix: Functional updater

```jsx
function FixedCounter() {
  const [count, setCount] = useState(0);

  const handleTripleIncrement = () => {
    // ✅ React passes the pending state to each callback in the batch
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    // Result: count correctly becomes 3
  };

  return <button onClick={handleTripleIncrement}>Add 3 (Works)</button>;
}
```

> 💡 **Best Practice:** Use the functional updater whenever the next state is calculated from the prior state, or inside callbacks, `setTimeout`, and effect closures where values might become stale.

---

### Lazy Initial State

If computing the initial state is expensive (e.g., reading from `localStorage`, parsing JSON, or running a heavy calculation), pass a function to `useState`.

#### ❌ Runs on every single render:
```jsx
// ❌ localStorage.getItem & JSON.parse runs on EVERY re-render!
const [user, setUser] = useState(JSON.parse(localStorage.getItem('user') || '{}'));
```

#### ✅ Runs ONLY on initial mount:
```jsx
// ✅ Function is only executed once during mounting
const [user, setUser] = useState(() => {
  try {
    const saved = localStorage.getItem('user');
    return saved ? JSON.parse(saved) : { name: 'Guest', loggedIn: false };
  } catch (err) {
    return { name: 'Guest', loggedIn: false };
  }
});
```

---

## Updating Objects in State

### Immutability Principle

React detects changes using **reference equality** (`Object.is(oldState, newState)`). If you mutate an object in place, the memory address does not change, and React will skip re-rendering!

```jsx
// ❌ MUTATION: React will NOT detect this change!
const [user, setUser] = useState({ name: 'Alex', age: 25 });
user.name = 'Jordan';
setUser(user); // Same reference! Nothing re-renders.

// ✅ IMMUTABLE UPDATE: Create a new object reference
setUser({ ...user, name: 'Jordan' });
```

---

### Shallow Object Updates

Use the object spread operator (`...`) to copy existing properties and overwrite the ones you want to update:

```jsx
import { useState } from 'react';

function ProfileForm() {
  const [profile, setProfile] = useState({
    username: 'abhineshgour',
    email: 'abhi@example.com',
    role: 'developer'
  });

  const handleChange = (e) => {
    const { name, value } = e.target;
    // Copy all previous fields, override the dynamic key
    setProfile(prev => ({
      ...prev,
      [name]: value
    }));
  };

  return (
    <form>
      <input 
        name="username" 
        value={profile.username} 
        onChange={handleChange} 
      />
      <input 
        name="email" 
        value={profile.email} 
        onChange={handleChange} 
      />
    </form>
  );
}
```

---

### Nested Object Updates

Spread operators only copy one level deep (shallow copy). Nested objects must be spread at every level.

```jsx
const [employee, setEmployee] = useState({
  id: 101,
  name: 'Devon',
  contact: {
    email: 'devon@company.com',
    address: {
      city: 'Bangalore',
      zip: '560001'
    }
  }
});

// Update city inside deeply nested address:
const updateCity = (newCity) => {
  setEmployee(prev => ({
    ...prev,
    contact: {
      ...prev.contact,
      address: {
        ...prev.contact.address,
        city: newCity
      }
    }
  }));
};
```

---

### Deep Clones & Helpers

When nesting gets 3+ levels deep, manual spreading becomes brittle and hard to read. You can use modern native JS or libraries:

#### 1. Native `structuredClone()` (Node 17+, modern browsers):
```jsx
const updateDeep = (newCity) => {
  setEmployee(prev => {
    const clone = structuredClone(prev);
    clone.contact.address.city = newCity;
    return clone;
  });
};
```

#### 2. Using `immer` (recommended for complex state trees):
```jsx
import { produce } from 'immer';

const updateWithImmer = (newCity) => {
  setEmployee(produce(draft => {
    draft.contact.address.city = newCity;
  }));
};
```

---

## Updating Arrays in State

Never use mutating array methods like `.push()`, `.pop()`, `.shift()`, `.unshift()`, `.splice()`, or `.sort()`. Instead, use non-mutating methods that return new array instances (`concat`, `filter`, `map`, `slice`, spread `[...]`).

| Operation | ❌ Mutating (Avoid) | ✅ Non-mutating (Use) |
|:---|:---|:---|
| **Add** | `push()`, `unshift()` | `[...arr, newItem]` or `[newItem, ...arr]` |
| **Remove** | `pop()`, `shift()`, `splice()` | `arr.filter(item => item.id !== id)` |
| **Update** | `arr[index] = val` | `arr.map(item => item.id === id ? { ...item, ...updates } : item)` |
| **Sort** | `arr.sort()` | `[...arr].sort(...)` |
| **Reverse** | `arr.reverse()` | `[...arr].reverse()` or `arr.toReversed()` |

### Complete Todo Array Example

```jsx
import { useState } from 'react';

function TodoManager() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'Master React state', completed: false },
    { id: 2, text: 'Learn useEffect traps', completed: true }
  ]);

  // 1. ADD
  const addTodo = (text) => {
    const newTodo = { id: Date.now(), text, completed: false };
    setTodos(prev => [...prev, newTodo]);
  };

  // 2. REMOVE
  const removeTodo = (id) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  };

  // 3. UPDATE / TOGGLE
  const toggleTodo = (id) => {
    setTodos(prev => prev.map(todo => 
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  };

  // 4. SORT (copy first!)
  const sortAlphabetically = () => {
    setTodos(prev => [...prev].sort((a, b) => a.text.localeCompare(b.text)));
  };

  return (
    <div>
      <button onClick={() => addTodo('New Task ' + Math.random().toString().slice(2, 6))}>
        Add Task
      </button>
      <button onClick={sortAlphabetically}>Sort Tasks</button>
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>
            <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
              {todo.text}
            </span>
            <button onClick={() => toggleTodo(todo.id)}>Toggle</button>
            <button onClick={() => removeTodo(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## Component Lifecycle (Conceptual)

Every component instance goes through three primary lifecycle phases:

```
  ┌─────────────────┐
  │    1. MOUNT     │  Component is inserted into DOM for the first time.
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │   2. UPDATE     │  State or props change; re-render occurs; DOM diffed & updated.
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │   3. UNMOUNT    │  Component is removed from DOM (navigated away, conditionally hidden).
  └─────────────────┘
```

### Class Lifecycle vs. Hook Model

Historically, React components were ES6 classes using imperative lifecycle hooks. Modern React functional components use hooks that synchronize based on data dependencies rather than arbitrary lifecycle events:

```
Class Component Lifecycles:
┌──────────────────────────┬──────────────────────────┬─────────────────────────┐
│ componentDidMount        │ componentDidUpdate       │ componentWillUnmount    │
└────────────┬─────────────┴────────────┬─────────────┴────────────┬────────────┘
             │                          │                          │
             ▼                          ▼                          ▼
Functional Component (`useEffect`):
┌───────────────────────────────────────────────────────────────────────────────┐
│ useEffect(() => {                                                             │
│   // Runs after mount AND after updates (when dependencies change)            │
│   return () => {                                                              │
│     // Cleanup runs BEFORE the effect re-runs AND upon unmount                │
│   };                                                                          │
│ }, [dependencies]);                                                           │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## The `useEffect` Hook

### Mental Model: Synchronization

`useEffect` allows you to synchronize your component with an **external system** — such as:
- Browser APIs (`localStorage`, `window.addEventListener`, document title)
- Network calls (fetching data over REST/GraphQL)
- Timers (`setInterval`, `setTimeout`)
- Third-party non-React libraries (Leaflet maps, Chart.js)

> ⚠️ `useEffect` is **not** meant for calculating state derivations that can be done during render! If a value can be computed from existing props or state, compute it directly in the component body.

---

### The Dependency Array Demystified

The second argument to `useEffect` defines *when* the effect should execute:

```jsx
// 1. NO DEPENDENCY ARRAY
// Runs after EVERY single render (initial mount + every update)
useEffect(() => {
  console.log('Component rendered!');
});

// 2. EMPTY DEPENDENCY ARRAY []
// Runs ONLY ONCE after initial mount
useEffect(() => {
  console.log('Component mounted in DOM!');
}, []);

// 3. WITH DEPENDENCIES [a, b]
// Runs on mount AND whenever 'a' OR 'b' changes reference
useEffect(() => {
  console.log(`Dependency updated: ${a}, ${b}`);
}, [a, b]);
```

---

### The Cleanup Function

If your effect returns a function, React calls that function:
1. **Before the effect runs again** (on subsequent updates).
2. **When the component unmounts** from the DOM.

Cleanup functions are critical for preventing memory leaks, duplicate event handlers, and rogue timers.

```jsx
import { useState, useEffect } from 'react';

function WindowWidthTracker() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);

    // 1. Setup subscription
    window.addEventListener('resize', handleResize);

    // 2. Teardown subscription
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []); // Runs once on mount; cleans up on unmount

  return <p>Window Width: {width}px</p>;
}
```

---

### Common Use Cases

#### 1. Page Title Synchronization
```jsx
useEffect(() => {
  document.title = `Inbox (${unreadCount} new messages)`;
}, [unreadCount]);
```

#### 2. Interval Timer with Proper Cleanup
```jsx
function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const intervalId = setInterval(() => {
      // ✅ Use functional updater to avoid stale closure on seconds
      setSeconds(prev => prev + 1);
    }, 1000);

    return () => clearInterval(intervalId); // Cleanup!
  }, []);

  return <div>Elapsed: {seconds}s</div>;
}
```

---

## Common `useEffect` Pitfalls & Traps

### 1. Infinite Render Loops

Setting state inside an effect that depends on that very state (or has no dependency array) creates an endless loop.

```jsx
// ❌ INFINITE LOOP!
function BrokenComponent() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    // This updates count -> triggers re-render -> runs effect -> updates count...
    setCount(count + 1);
  }); // No dependency array!
}
```

---

### 2. Stale Closures & Missing Dependencies

JavaScript closures capture variables from their outer scope at execution time. If you omit a dependency, the effect will remember the variable's value from the initial render forever!

```jsx
// ❌ Stale Closure: `count` is trapped at 0 forever
useEffect(() => {
  const timer = setInterval(() => {
    setCount(count + 1); // Reads initial 0, sets to 1, next time still reads 0!
  }, 1000);
  return () => clearInterval(timer);
}, []); // eslint warns: missing dependency `count`

// ✅ Option A: Add count to dependencies (resets timer every second)
useEffect(() => {
  const timer = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(timer);
}, [count]);

// ✅ Option B (Better): Use functional updater (timer persists uninterrupted)
useEffect(() => {
  const timer = setInterval(() => setCount(prev => prev + 1), 1000);
  return () => clearInterval(timer);
}, []);
```

---

### 3. Data Fetching Race Conditions

When searching or switching pages rapidly, slow responses from earlier requests can resolve *after* newer requests, overwriting fresh data with stale data.

```jsx
import { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let isCurrent = true; // Flag for cleanup

    async function loadUser() {
      const res = await fetch(`https://api.example.com/users/${userId}`);
      const data = await res.json();
      
      // Only update state if this request is still the active one!
      if (isCurrent) {
        setUser(data);
      }
    }

    loadUser();

    return () => {
      isCurrent = false; // Invalidates response if userId changed before request completed
    };
  }, [userId]);

  return <div>{user ? user.name : 'Loading...'}</div>;
}
```

Alternatively, use the native `AbortController`:
```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch(`https://api.example.com/users/${userId}`, { signal: controller.signal })
    .then(res => res.json())
    .then(data => setUser(data))
    .catch(err => {
      if (err.name !== 'AbortError') {
        console.error('Fetch error:', err);
      }
    });

  return () => controller.abort(); // Cancels active network request on unmount/re-run
}, [userId]);
```

---

### 4. Object and Array Dependencies

In JavaScript, `{}` !== `{}` and `[]` !== `[]`. Declaring an object or array literal inside the component body creates a new reference on every render, triggering effects endlessly if passed into dependencies.

```jsx
function SearchComponent({ query }) {
  // ❌ New object reference created on every single render!
  const options = { filter: query, page: 1 };

  useEffect(() => {
    fetchResults(options);
  }, [options]); // ❌ Runs every render because `options` is always a new reference!
}

// ✅ Fix 1: Pass primitive dependencies directly
useEffect(() => {
  fetchResults({ filter: query, page: 1 });
}, [query]);

// ✅ Fix 2: Memoize the object using useMemo (when necessary)
const options = useMemo(() => ({ filter: query, page: 1 }), [query]);
```

---

## `useLayoutEffect` vs `useEffect`

Both hooks have identical signatures, but they run at different points during the browser painting pipeline:

```
State change
    │
    ▼
React Renders (Virtual DOM)
    │
    ▼
React Mutates Real DOM
    │
    ├─► useLayoutEffect runs SYNCHRONOUSLY here (before browser repaints)
    │   (Blocks visual paint; perfect for DOM measurements/scroll adjustments)
    │
Browser Repaints Screen
    │
    └─► useEffect runs ASYNCHRONOUSLY here (after screen is painted)
        (Does not block UI; default choice for 99% of use cases)
```

### When to use `useLayoutEffect`:
- Measuring DOM element dimensions (e.g., `getBoundingClientRect()`) before rendering a tooltip so it doesn't flicker on screen.
- Scroll position adjustments before the user sees the page flash.
- Preventing visual layout shifts ("flashing" or "jitter").

---

## 🏋️ Practice Exercises

### Exercise 1: Stopwatch with Lap Times
Build a Stopwatch component with:
1. Start, Pause, and Reset buttons.
2. A "Lap" button that records the current timestamp into an array of laps without mutating state.
3. Accurate cleanup of the underlying `setInterval`.

### Exercise 2: Debounced Live Search
Build a live search input that:
1. Takes user input via `useState`.
2. Uses `useEffect` with a `setTimeout` of 400ms to simulate an API request.
3. Cancels the timer in the cleanup function if the user types again within 400ms.

### Exercise 3: Window Resize & Orientation Tracker
Create a custom tracker that:
1. Stores `{ width, height, isMobile }` where `isMobile` is `width < 768`.
2. Attaches a window `resize` listener and cleans it up properly upon unmount.

---

> **Previous:** [← 01 — React Fundamentals](./01-fundamentals.md)  
> **Next:** [03 — Hooks Deep Dive →](./03-hooks-deep-dive.md)
