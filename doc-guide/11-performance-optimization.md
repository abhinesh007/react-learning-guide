# 11 — Performance Optimization

> Master React rendering mechanics, profiling, memoization (`memo`, `useMemo`, `useCallback`), virtualization, concurrent transitions, and bundle trimming.

---

## Table of Contents

- [How React Renders (The Render Pipeline)](#how-react-renders-the-render-pipeline)
  - [Trigger → Render → Commit Pipeline](#trigger--render--commit-pipeline)
  - [Re-render vs. DOM Mutation](#re-render-vs-dom-mutation)
- [Diagnosing Bottlenecks with React DevTools Profiler](#diagnosing-bottlenecks-with-react-devtools-profiler)
  - [Flamegraph & Ranked Charts](#flamegraph--ranked-charts)
  - [Highlighting Updates in the Browser](#highlighting-updates-in-the-browser)
- [Skipping Re-renders with `React.memo`](#skipping-re-renders-with-reactmemo)
  - [Default Shallow Comparison](#default-shallow-comparison)
  - [Custom Comparison Functions (`arePropsEqual`)](#custom-comparison-functions-arepropsequal)
  - [When to Memoize vs. Premature Optimization](#when-to-memoize-vs-premature-optimization)
- [Optimizing Calculations with `useMemo`](#optimizing-calculations-with-usememo)
  - [Cost Thresholds: When is a calculation "expensive"?](#cost-thresholds-when-is-a-calculation-expensive)
  - [Filtering & Sorting Massive Datasets](#filtering--sorting-massive-datasets)
- [Preserving Callback Identity with `useCallback`](#preserving-callback-identity-with-usecallback)
  - [The Inline Function Recreation Problem](#the-inline-function-recreation-problem)
  - [Pairing `useCallback` with `React.memo`](#pairing-usecallback-with-reactmemo)
- [Concurrent React: Non-Blocking UI](#concurrent-react-non-blocking-ui)
  - [`useTransition` for Background Transitions](#usetransition-for-background-transitions)
  - [`useDeferredValue` for Snappy Input Fields](#usedeferredvalue-for-snappy-input-fields)
- [Virtualization (Rendering 100,000 Items)](#virtualization-rendering-100000-items)
  - [Why the DOM Chokes on Large Lists](#why-the-dom-chokes-on-large-lists)
  - [Implementing `@tanstack/react-virtual`](#implementing-tanstackreact-virtual)
  - [`react-window` Alternative](#react-window-alternative)
- [Code Splitting & Dynamic Imports](#code-splitting--dynamic-imports)
  - [Route-Level Splitting](#route-level-splitting)
  - [Component-Level & Modal Lazy Loading](#component-level--modal-lazy-loading)
- [Asset & Image Optimization](#asset--image-optimization)
- [Debouncing and Throttling](#debouncing-and-throttling)
- [Bundle Size Analysis](#bundle-size-analysis)
  - [Using `rollup-plugin-visualizer` with Vite](#using-rollup-plugin-visualizer-with-vite)
- [React Performance Audit Checklist](#react-performance-audit-checklist)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## How React Renders (The Render Pipeline)

### Trigger → Render → Commit Pipeline

Every UI update in React passes through three distinct phases:

```
  1. TRIGGER               2. RENDER                3. COMMIT
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│ State/Props │ ───────► │ React calls │ ───────► │ React writes│
│   Change    │          │ Components  │          │ only changed│
│             │          │ (Virtual DOM│          │ nodes to real│
│             │          │  diffing)   │          │ DOM screen  │
└─────────────┘          └─────────────┘          └─────────────┘
```

1. **Trigger:** A state variable changes, a Context Provider emits a new value, or a parent component re-renders.
2. **Render:** React invokes your component functions to build a new Virtual DOM tree and diffs it with the prior tree.
3. **Commit:** React updates the real browser DOM with the calculated differences.

---

### Re-render vs. DOM Mutation

> 💡 **Crucial Distinction:** A component **re-rendering** simply means running its JavaScript function. It does **NOT** mean the browser DOM nodes are recreated! If the output JSX is identical, React skips touching the real DOM completely.

However, running expensive component functions with 50 children dozens of times per second wastes CPU cycles and drops frame rates.

---

## Diagnosing Bottlenecks with React DevTools Profiler

Never optimize blindly. Always measure first using the **React DevTools Profiler**:

1. Open Chrome DevTools → Click the **Profiler** tab.
2. Click **Start Recording** (Blue circle) → Perform the user action (e.g. typing or clicking).
3. Click **Stop Recording**.

### Flamegraph & Ranked Charts
- **Flamegraph:** Visualizes the component hierarchy and highlights which components rendered (yellow/orange = took longer; grey = skipped render).
- **Ranked Chart:** Ranks components strictly by execution time, immediately showing your biggest bottlenecks at the top.
- Check the box: *"Record why each component rendered while profiling"*. DevTools will tell you: *"Props changed: [onClick]"* or *"Parent component rendered"*.

---

## Skipping Re-renders with `React.memo`

By default, when a parent component re-renders, **all of its children re-render recursively**, regardless of whether their props changed!

`React.memo` wraps a component so React skips re-rendering it if its incoming props are identical to the prior render.

### Default Shallow Comparison

```jsx
import { memo } from 'react';

// Skipped unless `user` or `onSelect` reference changes
export const UserCard = memo(function UserCard({ user, onSelect }) {
  console.log(`Rendered UserCard: ${user.name}`);
  return (
    <div className="card" onClick={() => onSelect(user.id)}>
      <h3>{user.name}</h3>
      <p>{user.title}</p>
    </div>
  );
});
```

---

### Custom Comparison Functions (`arePropsEqual`)

You can provide a second argument to customize the equality check:

```jsx
function arePropsEqual(prevProps, nextProps) {
  // Return true to SKIP re-render, false to re-render
  return (
    prevProps.user.id === nextProps.user.id &&
    prevProps.user.name === nextProps.user.name &&
    prevProps.isHighlighted === nextProps.isHighlighted
  );
}

export const OptimizedCard = memo(UserCard, arePropsEqual);
```

---

### When to Memoize vs. Premature Optimization

| ✅ Good Use Cases for `React.memo` | ❌ Bad / Useless Cases |
|:---|:---|
| Pure presentation components with lots of DOM elements | Cheap components (e.g., `<button>{label}</button>`) |
| Components that re-render frequently with the exact same props | Components whose props change on *every* render anyway |
| Large lists of items where only 1 item updates at a time | Components receiving new object/array literals without `useMemo` |

---

## Optimizing Calculations with `useMemo`

### Cost Thresholds: When is a calculation "expensive"?

Do not wrap simple additions or small array mappings in `useMemo`. The memory overhead of storing dependency arrays and running comparison checks often exceeds the nanoseconds saved.

Benchmark rule of thumb: If an algorithm takes **> 1-2 milliseconds** (e.g. sorting 5,000 records, calculating statistics, running graph algorithms), memoize it!

```jsx
// ❌ Premature: Primitive string lowercase takes 0.001ms
const formattedName = useMemo(() => name.toLowerCase(), [name]);

// ✅ Good: Filtering and sorting 10,000 records
const filteredRecords = useMemo(() => {
  return rawRecords
    .filter(r => r.active && r.score >= threshold)
    .sort((a, b) => b.score - a.score);
}, [rawRecords, threshold]);
```

---

## Preserving Callback Identity with `useCallback`

### The Inline Function Recreation Problem

Every time a component re-renders, any function defined inside it is recreated as a **brand-new object reference**:

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  // ⚠️ New function instance created on every single Parent render!
  const handleDelete = (id) => {
    deleteItem(id);
  };

  // Even though ExpensiveList is wrapped in React.memo,
  // it WILL re-render because handleDelete is a new reference every time!
  return <ExpensiveList onDelete={handleDelete} />;
}
```

---

### Pairing `useCallback` with `React.memo`

```jsx
import { useState, useCallback } from 'react';

function Parent() {
  const [count, setCount] = useState(0);

  // ✅ Same function reference preserved across renders
  const handleDelete = useCallback((id) => {
    deleteItem(id);
  }, []); // Dependencies

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Counter: {count}</button>
      <ExpensiveList onDelete={handleDelete} />
    </div>
  );
}
```

> 💡 `useCallback(fn, deps)` is strictly equivalent to `useMemo(() => fn, deps)`.

---

## Concurrent React: Non-Blocking UI

### `useTransition` for Background Transitions

Keeps input responses, button clicks, and typing at 60 FPS while expensive UI transitions happen in the background:

```jsx
import { useState, useTransition } from 'react';

export function SearchFilter() {
  const [inputVal, setInputVal] = useState('');
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleInputChange = (e) => {
    const nextVal = e.target.value;
    // 1. Urgent: Updates input instantly
    setInputVal(nextVal);

    // 2. Non-Urgent: Can be interrupted if user types another letter!
    startTransition(() => {
      setQuery(nextVal);
    });
  };

  return (
    <div>
      <input value={inputVal} onChange={handleInputChange} placeholder="Search..." />
      {isPending && <span className="spinner">Filtering results...</span>}
      <MassiveResultsTable query={query} />
    </div>
  );
}
```

---

### `useDeferredValue` for Snappy Input Fields

```jsx
import { useState, useDeferredValue } from 'react';

export function SearchBox() {
  const [text, setText] = useState('');
  // Defers recalculation until urgent renders (typing) finish
  const deferredQuery = useDeferredValue(text);

  return (
    <div>
      <input value={text} onChange={e => setText(e.target.value)} />
      <SlowList query={deferredQuery} />
    </div>
  );
}
```

---

## Virtualization (Rendering 100,000 Items)

### Why the DOM Chokes on Large Lists

Rendering 10,000 items into the DOM creates 10,000+ real DOM nodes. The browser runs out of memory and experiences severe layout thrashing during scroll events.

**Virtualization (Windowing)** renders only the 20-30 DOM elements currently visible in the user's viewport, swapping elements dynamically as the user scrolls:

```
┌─────────────────────────────────┐
│ Virtual Total Height: 50,000px  │
│                                 │
│ ┌─────────────────────────────┐ │
│ │ Screen Viewport (Visible)   │ │ ◄── Only these 10 items exist in DOM!
│ │ Item #201                   │ │
│ │ Item #202                   │ │
│ │ Item #203                   │ │
│ └─────────────────────────────┘ │
│                                 │
└─────────────────────────────────┘
```

---

### Implementing `@tanstack/react-virtual`

```bash
npm install @tanstack/react-virtual
```

```jsx
import { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

export function VirtualizedList({ items }) {
  const parentRef = useRef(null);

  const rowVirtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40, // Height in px of each row
    overscan: 5,           // Render 5 items above and below viewport
  });

  return (
    <div
      ref={parentRef}
      style={{ height: '400px', overflow: 'auto', border: '1px solid #ddd' }}
    >
      <div
        style={{
          height: `${rowVirtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {rowVirtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            Row #{virtualRow.index}: {items[virtualRow.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## Code Splitting & Dynamic Imports

### Route-Level Splitting

```jsx
import { lazy, Suspense } from 'react';

const AdminPortal = lazy(() => import('./pages/AdminPortal'));

<Suspense fallback={<p>Loading Admin...</p>}>
  <AdminPortal />
</Suspense>
```

### Component-Level & Modal Lazy Loading

Never bundle heavy components (e.g. Rich Text Editors, Chart Dashboards, Export Modals) in the initial bundle if they only appear after a button click:

```jsx
const RichTextEditor = lazy(() => import('./RichTextEditor'));

function PostEditor() {
  const [showEditor, setShowEditor] = useState(false);

  return (
    <div>
      <button onClick={() => setShowEditor(true)}>Open Editor</button>
      {showEditor && (
        <Suspense fallback={<div>Loading Editor Engine...</div>}>
          <RichTextEditor />
        </Suspense>
      )}
    </div>
  );
}
```

---

## Asset & Image Optimization

1. **Native Lazy Loading:** Add `loading="lazy"` to all non-hero images:
   ```jsx
   <img src={item.imageUrl} alt={item.name} loading="lazy" decoding="async" />
   ```
2. **Next-gen formats:** Prefer `.webp` or `.avif` over raw `.png` / `.jpg`.
3. **Explicit Dimensions:** Always supply `width` and `height` attributes to prevent Cumulative Layout Shifts (CLS).

---

## Debouncing and Throttling

- **Debounce:** Waits until input has paused for $N$ ms (Search autocomplete).
- **Throttle:** Limits execution to at most once every $N$ ms (Window scroll, mouse tracking).

```jsx
import { useState, useMemo } from 'react';
import debounce from 'lodash/debounce';

export function Search() {
  const [query, setQuery] = useState('');

  // Stable debounced handler
  const debouncedSearch = useMemo(
    () => debounce((text) => sendAnalytics(text), 400),
    []
  );

  const handleChange = (e) => {
    setQuery(e.target.value);
    debouncedSearch(e.target.value);
  };

  return <input value={query} onChange={handleChange} />;
}
```

---

## Bundle Size Analysis

### Using `rollup-plugin-visualizer` with Vite

```bash
npm install -D rollup-plugin-visualizer
```

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true, filename: 'bundle-stats.html' }),
  ],
});
```

Running `npm run build` will automatically generate an interactive visual map of all package sizes in your bundle, spotting heavy libraries (e.g., uncompressed `moment.js` or full `lodash`).

---

## React Performance Audit Checklist

- [ ] **Are heavy computations wrapped in `useMemo`?** (Only if cost > 1ms).
- [ ] **Are callbacks passed to memoized children wrapped in `useCallback`?**
- [ ] **Are routes and heavy non-initial modals lazily loaded via `React.lazy()`?**
- [ ] **Are lists over 100 items virtualized with `@tanstack/react-virtual`?**
- [ ] **Are images loaded with `loading="lazy"` and explicit dimensions?**
- [ ] **Are input filters using `useTransition` or `useDeferredValue` for responsiveness?**
- [ ] **Are large dependencies audited with a bundle visualizer?**

---

## 🏋️ Practice Exercises

### Exercise 1: Optimize a Lagging Table
Take a table rendering 2,000 rows with sorting and search filters:
- Profile re-renders in DevTools.
- Memoize row items with `React.memo` and stable action callbacks.
- Defer search filtering with `useDeferredValue`.

### Exercise 2: Virtualize a 10,000 Item List
Using `@tanstack/react-virtual`, create a scrollable list of 10,000 simulated log lines that renders smoothly at 60 FPS without memory consumption issues.

---

> **Previous:** [← 10 — Testing](./10-testing.md)  
> **Next:** [12 — TypeScript with React →](./12-typescript-with-react.md)
