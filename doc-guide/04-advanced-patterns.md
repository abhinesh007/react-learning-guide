# 04 — Advanced Component Patterns

> Architectural patterns used by design systems, enterprise libraries, and production teams to build reusable, resilient, and composable React components.

---

## Table of Contents

- [Higher-Order Components (HOCs)](#higher-order-components-hocs)
  - [Pattern Overview & Diagram](#pattern-overview--diagram)
  - [Implementing `withAuth`](#implementing-withauth)
  - [Implementing `withLoading`](#implementing-withloading)
  - [HOC Trade-offs & When to Use](#hoc-trade-offs--when-to-use)
- [Render Props](#render-props)
  - [Pattern Concept](#pattern-concept)
  - [Mouse Tracker Implementation](#mouse-tracker-implementation)
  - [Render Props vs Custom Hooks](#render-props-vs-custom-hooks)
- [Compound Components](#compound-components)
  - [Mental Model: Inversion of Control](#mental-model-inversion-of-control)
  - [Building a Compound `<Tabs>` Component](#building-a-compound-tabs-component)
  - [Building an Accessible `<Accordion>`](#building-an-accessible-accordion)
- [Controlled vs Uncontrolled Components](#controlled-vs-uncontrolled-components)
  - [Decision Matrix](#decision-matrix)
  - [Designing a Flexible Controlled `<Modal>`](#designing-a-flexible-controlled-modal)
- [React Portals (`createPortal`)](#react-portals-createportal)
  - [Escaping DOM Hierarchy](#escaping-dom-hierarchy)
  - [Event Bubbling Nuances Across Portals](#event-bubbling-nuances-across-portals)
- [Error Boundaries](#error-boundaries)
  - [Class-Based Error Boundaries](#class-based-error-boundaries)
  - [Lifecycle Methods: `getDerivedStateFromError` & `componentDidCatch`](#lifecycle-methods-getderivedstatefromerror--componentdidcatch)
  - [Using `react-error-boundary`](#using-react-error-boundary)
- [Suspense & Code Splitting](#suspense--code-splitting)
  - [`React.lazy()` Dynamic Imports](#reactlazy-dynamic-imports)
  - [`<Suspense>` Fallback Boundaries](#suspense-fallback-boundaries)
- [Context Module Pattern](#context-module-pattern)
- [State Initializer / Reset Pattern](#state-initializer--reset-pattern)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## Higher-Order Components (HOCs)

### Pattern Overview & Diagram

A **Higher-Order Component** is a pure function that takes a component as an argument and returns an enhanced version of that component:

$$\text{EnhancedComponent} = f(\text{WrappedComponent})$$

```
                      ┌──────────────────────┐
                      │   WrappedComponent   │
                      └──────────┬───────────┘
                                 │ Passed as parameter
                                 ▼
┌─────────────────────────────────────────────────────────────┐
│ Higher-Order Component Function                             │
│ - Injects additional props                                  │
│ - Adds conditional rendering or checks (Auth, Loading)      │
│ - Subscribes to events or data streams                      │
└────────────────────────────────┬────────────────────────────┘
                                 │ Returns
                                 ▼
                      ┌──────────────────────┐
                      │   EnhancedComponent  │
                      └──────────────────────┘
```

---

### Implementing `withAuth`

Protects a component by verifying authentication status before rendering.

```jsx
import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';

export function withAuth(WrappedComponent) {
  return function AuthenticatedComponent(props) {
    const navigate = useNavigate();
    const token = localStorage.getItem('auth_token');

    useEffect(() => {
      if (!token) {
        navigate('/login', { replace: true });
      }
    }, [token, navigate]);

    if (!token) {
      return <div>Redirecting to authentication...</div>;
    }

    // Forward all incoming props to the wrapped component
    return <WrappedComponent {...props} token={token} />;
  };
}

// Usage:
function Dashboard({ token }) {
  return <h1>Secure Dashboard (Token: {token})</h1>;
}

export const ProtectedDashboard = withAuth(Dashboard);
```

---

### Implementing `withLoading`

Abstracts asynchronous data loading indicators away from presentation components:

```jsx
export function withLoading(WrappedComponent) {
  return function WithLoadingComponent({ isLoading, ...props }) {
    if (isLoading) {
      return (
        <div className="skeleton-loader" aria-busy="true">
          <div className="spinner" />
          <p>Fetching content...</p>
        </div>
      );
    }
    return <WrappedComponent {...props} />;
  };
}

// Usage:
function UserList({ users }) {
  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}

export const UserListWithLoader = withLoading(UserList);
```

---

### HOC Trade-offs & When to Use

| Advantages | Drawbacks |
|:---|:---|
| Reusable across multiple legacy or third-party components | "Wrapper Hell" (deeply nested component trees in DevTools) |
| Non-invasive enhancement without altering source code | Prop collisions (if HOC overrides a prop with the same name) |
| Well-understood across older codebases | Typescript typing for HOCs can be complex |

> 💡 **Modern Rule:** In modern React, custom hooks have replaced 90% of HOC use cases. However, HOCs remain useful when integrating third-party libraries, legacy class components, or cross-cutting route wrappers.

---

## Render Props

### Pattern Concept

A **Render Prop** is a technique for sharing code between React components using a prop whose value is a function that returns JSX. Instead of hardcoding what to render, the component delegates the rendering to the consumer.

```jsx
<DataProvider render={(data) => <h1>{data.title}</h1>} />
```

---

### Mouse Tracker Implementation

```jsx
import { useState } from 'react';

// Reusable logic component
export function MouseTracker({ children }) {
  const [coords, setCoords] = useState({ x: 0, y: 0 });

  const handleMouseMove = (event) => {
    setCoords({ x: event.clientX, y: event.clientY });
  };

  return (
    <div style={{ height: '300px', border: '1px dashed #ccc' }} onMouseMove={handleMouseMove}>
      {/* Invoke children or render prop function */}
      {typeof children === 'function' ? children(coords) : null}
    </div>
  );
}

// Consumer component:
export function MouseApp() {
  return (
    <MouseTracker>
      {({ x, y }) => (
        <p>The mouse pointer is currently at ({x}, {y})</p>
      )}
    </MouseTracker>
  );
}
```

---

### Render Props vs Custom Hooks

| Feature | Render Props | Custom Hooks |
|:---|:---|:---|
| **Syntax** | JSX component with function child | Direct function call in component body |
| **Component Tree** | Adds visual hierarchy nodes | Zero DOM or component nesting |
| **Composition** | Nested callbacks can create "callback hell" | Linear statements top-to-bottom |
| **Dynamic JSX** | Excellent for conditional slot rendering | Purely stateful logic (returns data/callbacks) |

---

## Compound Components

### Mental Model: Inversion of Control

Compound components work together as a cohesive unit with implicit state sharing. Think of native HTML: `<select>` and `<option>`. You don't pass an array of options to `<select options={...} />`; instead, you compose them declaratively:

```html
<select>
  <option value="1">One</option>
  <option value="2">Two</option>
</select>
```

In React, we implement this pattern using **Context API** combined with subcomponents attached as static properties.

---

### Building a Compound `<Tabs>` Component

```jsx
import { createContext, useContext, useState } from 'react';

// 1. Context for compound coordination
const TabsContext = createContext(null);

function useTabsContext() {
  const context = useContext(TabsContext);
  if (!context) {
    throw new Error('Tabs subcomponents must be rendered within a <Tabs> container');
  }
  return context;
}

// 2. Main Container Component
export function Tabs({ defaultIndex = 0, children }) {
  const [activeTab, setActiveTab] = useState(defaultIndex);

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs-container">{children}</div>
    </TabsContext.Provider>
  );
}

// 3. Tab List Header
function TabList({ children }) {
  return <div className="tab-buttons-row" role="tablist">{children}</div>;
}

// 4. Tab Trigger Button
function TabTrigger({ index, children }) {
  const { activeTab, setActiveTab } = useTabsContext();
  const isActive = activeTab === index;

  return (
    <button
      role="tab"
      aria-selected={isActive}
      className={`tab-btn ${isActive ? 'active' : ''}`}
      onClick={() => setActiveTab(index)}
    >
      {children}
    </button>
  );
}

// 5. Tab Panels Container
function TabPanels({ children }) {
  return <div className="tab-panels-content">{children}</div>;
}

// 6. Individual Tab Panel
function TabPanel({ index, children }) {
  const { activeTab } = useTabsContext();
  if (activeTab !== index) return null;

  return (
    <div role="tabpanel" className="tab-panel">
      {children}
    </div>
  );
}

// Attach subcomponents to the main Tabs component
Tabs.List = TabList;
Tabs.Trigger = TabTrigger;
Tabs.Panels = TabPanels;
Tabs.Panel = TabPanel;
```

#### Consumer Usage:
```jsx
export function UserProfileTabs() {
  return (
    <Tabs defaultIndex={0}>
      <Tabs.List>
        <Tabs.Trigger index={0}>Account</Tabs.Trigger>
        <Tabs.Trigger index={1}>Security</Tabs.Trigger>
        <Tabs.Trigger index={2}>Billing</Tabs.Trigger>
      </Tabs.List>

      <Tabs.Panels>
        <Tabs.Panel index={0}>Account settings & details</Tabs.Panel>
        <Tabs.Panel index={1}>Two-factor authentication settings</Tabs.Panel>
        <Tabs.Panel index={2}>Payment methods & invoices</Tabs.Panel>
      </Tabs.Panels>
    </Tabs>
  );
}
```

---

### Building an Accessible `<Accordion>`

```jsx
import { createContext, useContext, useState } from 'react';

const AccordionContext = createContext(null);

export function Accordion({ allowMultiple = false, children }) {
  const [openItems, setOpenItems] = useState(new Set());

  const toggleItem = (id) => {
    setOpenItems(prev => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        if (!allowMultiple) next.clear();
        next.add(id);
      }
      return next;
    });
  };

  return (
    <AccordionContext.Provider value={{ openItems, toggleItem }}>
      <div className="accordion-root">{children}</div>
    </AccordionContext.Provider>
  );
}

Accordion.Item = function AccordionItem({ id, title, children }) {
  const { openItems, toggleItem } = useContext(AccordionContext);
  const isOpen = openItems.has(id);

  return (
    <div className="accordion-item" style={{ borderBottom: '1px solid #e5e7eb' }}>
      <button
        className="accordion-header"
        onClick={() => toggleItem(id)}
        aria-expanded={isOpen}
        style={{ width: '100%', textAlign: 'left', padding: '12px', fontWeight: 600 }}
      >
        {title} <span>{isOpen ? '▲' : '▼'}</span>
      </button>
      {isOpen && (
        <div className="accordion-body" style={{ padding: '12px' }}>
          {children}
        </div>
      )}
    </div>
  );
};
```

---

## Controlled vs Uncontrolled Components

### Decision Matrix

| Characteristic | Controlled Component | Uncontrolled Component |
|:---|:---|:---|
| **Data Source** | React state (`useState`) | Real DOM node (`useRef`) |
| **Instant Validation** | Yes, on every keystroke | No, only on submit or blur |
| **Performance** | Triggers re-render per change | Zero React re-renders during editing |
| **Integration** | Natural with React design systems | Natural with third-party DOM plugins |
| **Code Verbosity** | Needs `value` + `onChange` | Single `ref` declaration |

---

### Designing a Flexible Controlled `<Modal>`

```jsx
import { useEffect } from 'react';
import { createPortal } from 'react-dom';

export function Modal({ isOpen, onClose, title, children }) {
  // Close on 'Escape' key
  useEffect(() => {
    const handleKeyDown = (e) => {
      if (e.key === 'Escape' && isOpen) {
        onClose();
      }
    };
    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div className="modal-backdrop" onClick={onClose} style={backdropStyle}>
      <div 
        className="modal-content" 
        onClick={(e) => e.stopPropagation()} // Stop bubbling to backdrop
        style={contentStyle}
      >
        <div className="modal-header" style={{ display: 'flex', justifyContent: 'space-between' }}>
          <h3>{title}</h3>
          <button onClick={onClose}>&times;</button>
        </div>
        <div className="modal-body">{children}</div>
      </div>
    </div>,
    document.body // Rendered directly at the top of document.body
  );
}

const backdropStyle = {
  position: 'fixed',
  top: 0,
  left: 0,
  width: '100vw',
  height: '100vh',
  backgroundColor: 'rgba(0, 0, 0, 0.5)',
  display: 'flex',
  justifyContent: 'center',
  alignItems: 'center',
  zIndex: 1000,
};

const contentStyle = {
  background: '#fff',
  borderRadius: '8px',
  padding: '24px',
  minWidth: '400px',
  maxWidth: '90%',
};
```

---

## React Portals (`createPortal`)

### Escaping DOM Hierarchy

Normally, a component renders directly inside its parent DOM node. However, overlays like modals, dropdown tooltips, and floating banners often break when wrapped inside a parent that has `overflow: hidden`, `z-index`, or `transform` applied.

`createPortal(children, domNode)` renders elements into any valid DOM node anywhere on the page while keeping the component conceptually in the same React tree!

```jsx
import { createPortal } from 'react-dom';

function FloatingNotification({ message }) {
  return createPortal(
    <div className="toast-notification">
      {message}
    </div>,
    document.getElementById('portal-root') || document.body
  );
}
```

---

### Event Bubbling Nuances Across Portals

Even though a portal element is mounted elsewhere in the real HTML DOM, **events bubble up the React tree, NOT the DOM tree**.

```jsx
function ParentWrapper() {
  const handleParentClick = () => {
    console.log('Parent received click event!');
  };

  return (
    <div onClick={handleParentClick} className="parent-box">
      <h2>Parent Container</h2>
      {/* Even though ChildPortal is in document.body, clicking it fires handleParentClick! */}
      <ChildPortal />
    </div>
  );
}

function ChildPortal() {
  return createPortal(
    <button onClick={() => console.log('Button clicked!')}>
      Click inside Portal
    </button>,
    document.body
  );
}
```

---

## Error Boundaries

React components can crash when an uncaught JavaScript error occurs during rendering, lifecycle execution, or constructor execution. In React, an unhandled error unmounts the entire component tree, leaving a blank white screen.

**Error Boundaries** catch JavaScript errors anywhere in their child component tree, log those errors, and display a fallback UI instead of crashing the entire application.

> ⚠️ As of React 19, Error Boundaries must still be authored as **Class Components** (or by using the `react-error-boundary` package).

---

### Class-Based Error Boundaries

```jsx
import React, { Component } from 'react';

export class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  // 1. Update state so next render shows fallback UI
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  // 2. Catch and log error metrics (Sentry, Datadog, etc.)
  componentDidCatch(error, errorInfo) {
    console.error('ErrorBoundary caught an error:', error, errorInfo);
  }

  handleReset = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError) {
      if (this.props.fallback) {
        return this.props.fallback({
          error: this.state.error,
          reset: this.handleReset,
        });
      }

      return (
        <div role="alert" style={{ padding: '24px', border: '1px solid red', borderRadius: '8px' }}>
          <h2>⚠️ Something went wrong</h2>
          <p style={{ color: 'red' }}>{this.state.error?.message}</p>
          <button onClick={this.handleReset}>Try Again</button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

---

### Using `react-error-boundary`

The popular `react-error-boundary` library provides hooks and ergonomic wrappers:

```bash
npm install react-error-boundary
```

```jsx
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }) {
  return (
    <div role="alert" className="error-card">
      <p>Error: {error.message}</p>
      <button onClick={resetErrorBoundary}>Recover & Retry</button>
    </div>
  );
}

export function App() {
  return (
    <ErrorBoundary
      FallbackComponent={ErrorFallback}
      onReset={() => {
        // Reset component or app state here
      }}
    >
      <MainApplicationContent />
    </ErrorBoundary>
  );
}
```

---

## Suspense & Code Splitting

Code splitting breaks large JavaScript bundles into smaller chunks loaded on-demand over the network when needed.

### `React.lazy()` Dynamic Imports

```jsx
import { lazy, Suspense } from 'react';

// Dynamic lazy imports
const AnalyticsDashboard = lazy(() => import('./AnalyticsDashboard'));
const SettingsModal = lazy(() => import('./SettingsModal'));

export function App() {
  return (
    <div>
      <h1>Application Dashboard</h1>

      {/* Suspense boundary renders fallback during network download */}
      <Suspense fallback={<div className="skeleton-placeholder">Loading charts...</div>}>
        <AnalyticsDashboard />
      </Suspense>
    </div>
  );
}
```

---

## Context Module Pattern

Avoid exposing raw Context or setState functions directly to components. Instead, encapsulate the context, provider, and domain-specific helper actions inside a dedicated module:

```jsx
// auth-context.jsx
import { createContext, useContext, useReducer } from 'react';

const AuthContext = createContext(null);

function authReducer(state, action) {
  switch (action.type) {
    case 'LOGIN_SUCCESS':
      return { user: action.payload, isAuthenticated: true };
    case 'LOGOUT':
      return { user: null, isAuthenticated: false };
    default:
      return state;
  }
}

export function AuthProvider({ children }) {
  const [state, dispatch] = useReducer(authReducer, { user: null, isAuthenticated: false });
  return <AuthContext.Provider value={{ state, dispatch }}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within an AuthProvider');
  return context;
}

// Named action helpers decouple consumer components from dispatch actions
export async function loginUser(dispatch, credentials) {
  try {
    const res = await fetch('/api/login', { method: 'POST', body: JSON.stringify(credentials) });
    const user = await res.json();
    dispatch({ type: 'LOGIN_SUCCESS', payload: user });
    return user;
  } catch (err) {
    console.error('Login failed', err);
    throw err;
  }
}

export function logoutUser(dispatch) {
  dispatch({ type: 'LOGOUT' });
}
```

---

## State Initializer / Reset Pattern

When you need to completely reset a component's internal state to its initial state, you have two patterns:

### Pattern A: Pass a dynamic `key`
Changing the `key` prop forces React to unmount the existing component and remount a brand-new instance from scratch with freshly initialized state.

```jsx
export function FormContainer() {
  const [version, setVersion] = useState(0);

  const resetAllFormState = () => {
    setVersion(v => v + 1); // Changing key causes complete clean remount!
  };

  return (
    <div>
      <ComplexForm key={version} />
      <button onClick={resetAllFormState}>Reset Form Completely</button>
    </div>
  );
}
```

---

## 🏋️ Practice Exercises

### Exercise 1: Compound `<Select>` and `<Option>`
Build a fully styled compound `<Select>` component that includes:
- `<Select>` container managing currently selected value.
- `<Select.Trigger>` showing the active selection or placeholder.
- `<Select.Options>` dropdown container.
- `<Select.Option value="...">` items that close the dropdown on selection.

### Exercise 2: Isolated Widget Error Boundary
Wrap individual dashboard widgets in their own Error Boundaries so that if one chart fails to parse data, the rest of the dashboard remains operational with only that widget showing a localized fallback.

---

> **Previous:** [← 03 — Hooks Deep Dive](./03-hooks-deep-dive.md)  
> **Next:** [05 — State Management →](./05-state-management.md)
