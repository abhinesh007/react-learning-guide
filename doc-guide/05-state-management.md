# 05 — State Management

> Complete architectural guide to React state management: Context API, Redux Toolkit, Zustand, and atomic state libraries (Jotai, Recoil) with clear decision frameworks.

---

## Table of Contents

- [When Do You Need Global State Management?](#when-do-you-need-global-state-management)
  - [The Prop Drilling Problem](#the-prop-drilling-problem)
  - [The 4 Categories of Application State](#the-4-categories-of-application-state)
  - [Decision Tree: Local vs Global](#decision-tree-local-vs-global)
- [Context API (Built-in Solution)](#context-api-built-in-solution)
  - [Implementation Patterns](#implementation-patterns)
  - [Production Auth Context Example](#production-auth-context-example)
  - [Context Pitfalls: Performance & Re-render Storms](#context-pitfalls-performance--re-render-storms)
  - [Context Optimization Strategies](#context-optimization-strategies)
- [Redux Toolkit (Industry Standard)](#redux-toolkit-industry-standard)
  - [Core Principles: Store, Slices, Actions, Reducers](#core-principles-store-slices-actions-reducers)
  - [Setting up Store & Slices](#setting-up-store--slices)
  - [Consuming State: `useSelector` & `useDispatch`](#consuming-state-useselector--usedispatch)
  - [Async Operations with `createAsyncThunk`](#async-operations-with-createasyncthunk)
  - [RTK Query Introduction](#rtk-query-introduction)
  - [When to Use Redux Toolkit](#when-to-use-redux-toolkit)
- [Zustand (Lightweight & Modern)](#zustand-lightweight--modern)
  - [Why Zustand is Dominating Modern React](#why-zustand-is-dominating-modern-react)
  - [Creating Stores & Atomic Selectors](#creating-stores--atomic-selectors)
  - [Async Actions inside Zustand](#async-actions-inside-zustand)
  - [Middleware: `persist`, `devtools`, `immer`](#middleware-persist-devtools-immer)
  - [Zustand vs Redux Toolkit](#zustand-vs-redux-toolkit)
- [Jotai (Atomic State)](#jotai-atomic-state)
  - [The Atomic Model (`atom` & `useAtom`)](#the-atomic-model-atom--useatom)
  - [Derived & Computed Atoms](#derived--computed-atoms)
  - [When to Choose Jotai](#when-to-choose-jotai)
- [Recoil (Brief Overview)](#recoil-brief-overview)
- [State Management Comparison Matrix](#state-management-comparison-matrix)
- [Architecture & Selection Decision Guide](#architecture--selection-decision-guide)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## When Do You Need Global State Management?

### The Prop Drilling Problem

When a state value owned by a root component is needed by a child nested 5 levels down, passing that prop through intermediate components that don't need it creates **prop drilling**.

```
    [App] (owns 'user' state)
      │
      ▼
   [Layout] (passes 'user' down without using it)
      │
      ▼
   [Sidebar] (passes 'user' down without using it)
      │
      ▼
   [UserMenu] (passes 'user' down without using it)
      │
      ▼
   [AvatarBadge] (finally displays user.avatarUrl!)
```

Prop drilling makes refactoring difficult and couples unrelated components together.

---

### The 4 Categories of Application State

Not all state belongs in a global store. Modern React architectures divide state into 4 distinct categories:

| State Type | Description | Best Tool / Location | Examples |
|:---|:---|:---|:---|
| **1. UI State** | Ephemeral, component-specific display states | `useState`, `useReducer` | Modal open/close, accordion expansion, active tab |
| **2. Server State** | Remote data fetched from an API (cached, async) | TanStack Query, SWR, RTK Query | User profiles, product listings, notifications |
| **3. URL State** | State preserved in the browser address bar | React Router (`useSearchParams`, `useParams`) | Pagination (`?page=2`), filters, active sorting |
| **4. Global Client State** | Client-only shared state needed across disparate routes | Zustand, Context API, Redux Toolkit | Shopping cart, user session, dark mode, audio player |

> ⚠️ **Key Realization:** If you use **TanStack Query** for server data and **React Router** for URL search parameters, you rarely need a massive global client store!

---

### Decision Tree: Local vs Global

```
Does multiple unrelated branches of the UI need this state?
├── NO  ──► Keep it LOCAL: useState / useReducer
└── YES ──► Is it fetched from a remote server/API?
             ├── YES ──► Use Server Cache: TanStack Query / SWR / RTK Query
             └── NO  ──► Is it navigation/filter/search related?
                          ├── YES ──► Store in URL: useSearchParams
                          └── NO  ──► Global Client Store: Zustand / Context / Redux
```

---

## Context API (Built-in Solution)

### Implementation Patterns

React's built-in Context API consists of three parts:
1. `createContext(defaultValue)`
2. `<Context.Provider value={...}>`
3. `useContext(Context)`

---

### Production Auth Context Example

```jsx
import { createContext, useContext, useState, useEffect, useMemo } from 'react';

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // Validate stored token on mount
    const token = localStorage.getItem('access_token');
    if (token) {
      fetch('/api/auth/me', { headers: { Authorization: `Bearer ${token}` } })
        .then(res => res.json())
        .then(data => setUser(data))
        .catch(() => localStorage.removeItem('access_token'))
        .finally(() => setIsLoading(false));
    } else {
      setIsLoading(false);
    }
  }, []);

  const login = async (credentials) => {
    const res = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials),
    });
    const data = await res.json();
    localStorage.setItem('access_token', data.token);
    setUser(data.user);
    return data.user;
  };

  const logout = () => {
    localStorage.removeItem('access_token');
    setUser(null);
  };

  // ✅ Memoize context value to prevent unneeded child re-renders
  const contextValue = useMemo(() => ({
    user,
    isAuthenticated: Boolean(user),
    isLoading,
    login,
    logout,
  }), [user, isLoading]);

  return (
    <AuthContext.Provider value={contextValue}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be consumed inside an <AuthProvider>');
  }
  return context;
}
```

---

### Context Pitfalls: Performance & Re-render Storms

When a Context Provider's `value` changes:
- **Every component consuming `useContext(MyContext)` will re-render!**
- It does **not** matter if that component only cares about one property in that object; any change to the provider value invalidates all consumers.

```jsx
// ❌ If count changes every 10ms, ComponentB will re-render constantly even though it only needs 'theme'!
<AppContext.Provider value={{ count, theme }}>
  <ComponentA /> {/* Uses count */}
  <ComponentB /> {/* Only uses theme! */}
</AppContext.Provider>
```

---

### Context Optimization Strategies

1. **Split Independent Contexts:** Group rapidly changing state into one context and slow-changing state into another (e.g., `ThemeContext` separate from `LiveMetricsContext`).
2. **Split State and Dispatch:** Provide state in one context and update callbacks/dispatch in another.
```jsx
const StateContext = createContext(null);
const DispatchContext = createContext(null);

export function CounterProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <DispatchContext.Provider value={dispatch}>
      <StateContext.Provider value={state}>
        {children}
      </StateContext.Provider>
    </DispatchContext.Provider>
  );
}
```

---

## Redux Toolkit (Industry Standard)

Redux Toolkit (RTK) is the official, opinionated, modern way to write Redux code. It eliminates the historical boilerplate of plain Redux (action constants, switch statements, immutable manual copying).

```
   ┌───────────────────────────────────────────────────┐
   │                    Redux Store                    │
   │   ┌───────────────┐           ┌───────────────┐   │
   │   │  authSlice    │           │   cartSlice   │   │
   │   └───────────────┘           └───────────────┘   │
   └─────────────▲───────────────────────────┬─────────┘
                 │                           │
         dispatch(action)          useSelector(state => ...)
                 │                           │
   ┌─────────────┴────────┐         ┌────────▼─────────┐
   │     Action Button    │         │ Component View   │
   └──────────────────────┘         └──────────────────┘
```

---

### Setting up Store & Slices

```bash
npm install @reduxjs/toolkit react-redux
```

#### 1. Create a Slice (`src/store/cartSlice.js`)
```javascript
import { createSlice } from '@reduxjs/toolkit';

const initialState = {
  items: [],
  isOpen: false,
};

export const cartSlice = createSlice({
  name: 'cart',
  initialState,
  reducers: {
    // RTK uses Immer internally: direct mutations are converted to immutable updates!
    addItem: (state, action) => {
      const existing = state.items.find(item => item.id === action.payload.id);
      if (existing) {
        existing.quantity += 1;
      } else {
        state.items.push({ ...action.payload, quantity: 1 });
      }
    },
    removeItem: (state, action) => {
      state.items = state.items.filter(item => item.id !== action.payload);
    },
    toggleCart: (state) => {
      state.isOpen = !state.isOpen;
    },
  },
});

export const { addItem, removeItem, toggleCart } = cartSlice.actions;
export default cartSlice.reducer;
```

#### 2. Configure Store (`src/store/store.js`)
```javascript
import { configureStore } from '@reduxjs/toolkit';
import cartReducer from './cartSlice';

export const store = configureStore({
  reducer: {
    cart: cartReducer,
  },
});
```

#### 3. Wrap Root with Provider
```jsx
import { Provider } from 'react-redux';
import { store } from './store/store';
import App from './App';

export default function Root() {
  return (
    <Provider store={store}>
      <App />
    </Provider>
  );
}
```

---

### Consuming State: `useSelector` & `useDispatch`

```jsx
import { useSelector, useDispatch } from 'react-redux';
import { addItem, removeItem, toggleCart } from './store/cartSlice';

export function CartWidget() {
  const dispatch = useDispatch();
  // Granular selector: only re-renders if cart.items changes
  const items = useSelector(state => state.cart.items);
  const isOpen = useSelector(state => state.cart.isOpen);

  const totalCount = items.reduce((sum, item) => sum + item.quantity, 0);

  return (
    <div>
      <button onClick={() => dispatch(toggleCart())}>
        Cart ({totalCount})
      </button>

      {isOpen && (
        <div className="cart-drawer">
          {items.map(item => (
            <div key={item.id}>
              <span>{item.name} x {item.quantity}</span>
              <button onClick={() => dispatch(removeItem(item.id))}>Delete</button>
            </div>
          ))}
          <button onClick={() => dispatch(addItem({ id: 1, name: 'Mechanical Keyboard' }))}>
            Add Sample Keyboard
          </button>
        </div>
      )}
    </div>
  );
}
```

---

### Async Operations with `createAsyncThunk`

```javascript
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

export const fetchProducts = createAsyncThunk(
  'products/fetchAll',
  async (_, { rejectWithValue }) => {
    try {
      const response = await fetch('/api/products');
      if (!response.ok) throw new Error('Server error');
      return await response.json();
    } catch (err) {
      return rejectWithValue(err.message);
    }
  }
);

const productsSlice = createSlice({
  name: 'products',
  initialState: { items: [], loading: false, error: null },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchProducts.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchProducts.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      .addCase(fetchProducts.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload;
      });
  },
});
```

---

### RTK Query Introduction

RTK Query is included inside `@reduxjs/toolkit` and eliminates the need for manual `createAsyncThunk` logic for HTTP fetching:

```javascript
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const productsApi = createApi({
  reducerPath: 'productsApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api/' }),
  endpoints: (builder) => ({
    getProducts: builder.query({
      query: () => 'products',
    }),
  }),
});

// Auto-generated hook
export const { useGetProductsQuery } = productsApi;
```

---

### When to Use Redux Toolkit

- **Large engineering teams** needing strict conventions, architecture consistency, and standardized debugging.
- **Complex shared state flows** with dozens of interrelated action dispatches.
- **Strict audit logs / undo-redo history** requirements where Redux DevTools time-travel debugging is critical.

---

## Zustand (Lightweight & Modern)

### Why Zustand is Dominating Modern React

- **Zero Boilerplate:** No Providers, no action wrappers, minimal setup.
- **No Context Re-render Bugs:** Components subscribe to exact slices of state via selector functions; components only re-render when their selected slice changes.
- **Works outside React:** You can read and write state inside non-component JS files (e.g., API interceptors).
- **Tiny Bundle:** ~1KB gzipped.

```bash
npm install zustand
```

---

### Creating Stores & Atomic Selectors

```javascript
// src/store/useCartStore.js
import { create } from 'zustand';

export const useCartStore = create((set) => ({
  items: [],
  isOpen: false,

  // Action methods defined directly in the store
  addItem: (product) => set((state) => {
    const existing = state.items.find(item => item.id === product.id);
    if (existing) {
      return {
        items: state.items.map(item =>
          item.id === product.id ? { ...item, quantity: item.quantity + 1 } : item
        ),
      };
    }
    return { items: [...state.items, { ...product, quantity: 1 }] };
  }),

  removeItem: (id) => set((state) => ({
    items: state.items.filter(item => item.id !== id),
  })),

  toggleCart: () => set((state) => ({ isOpen: !state.isOpen })),
  clearCart: () => set({ items: [] }),
}));
```

#### Consuming the Store with Selectors:
```jsx
import { useCartStore } from './useCartStore';

export function CartCounter() {
  // ✅ Only re-renders when items.length changes!
  const itemCount = useCartStore(state => state.items.length);
  return <div className="badge">{itemCount} items</div>;
}

export function CartToggle() {
  // ✅ Extracting an action never triggers a re-render!
  const toggleCart = useCartStore(state => state.toggleCart);
  return <button onClick={toggleCart}>Toggle Drawer</button>;
}
```

---

### Async Actions inside Zustand

Async logic is completely native; simply define an `async` function inside the store:

```javascript
export const useUserStore = create((set) => ({
  user: null,
  isLoading: false,
  error: null,

  fetchUser: async (id) => {
    set({ isLoading: true, error: null });
    try {
      const res = await fetch(`/api/users/${id}`);
      const data = await res.json();
      set({ user: data, isLoading: false });
    } catch (err) {
      set({ error: err.message, isLoading: false });
    }
  },
}));
```

---

### Middleware: `persist`, `devtools`, `immer`

Zustand features a composable middleware pipeline:

```javascript
import { create } from 'zustand';
import { persist, devtools } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

export const useSettingsStore = create(
  devtools(
    persist(
      immer((set) => ({
        theme: 'dark',
        volume: 80,
        notifications: { email: true, push: false },

        // Immer lets you write direct mutations
        toggleEmail: () => set((state) => {
          state.notifications.email = !state.notifications.email;
        }),

        setTheme: (theme) => set({ theme }),
      })),
      {
        name: 'app-settings-storage', // Key in localStorage
      }
    )
  )
);
```

---

### Zustand vs Redux Toolkit

| Feature | Zustand | Redux Toolkit |
|:---|:---|:---|
| **Bundle Size** | ~1.1 KB | ~12 KB |
| **Boilerplate** | Near Zero | Moderate |
| **Provider Requirement** | None (can use without Provider) | Requires `<Provider store={...}>` |
| **Redux DevTools** | Supported via middleware | Native support |
| **Immer Support** | Optional via middleware | Built-in by default |
| **Learning Curve** | Extremely low (5 mins) | Moderate |

---

## Jotai (Atomic State)

### The Atomic Model (`atom` & `useAtom`)

Jotai represents state as individual, composable units called **atoms** (inspired by Recoil):

```bash
npm install jotai
```

```jsx
import { atom, useAtom } from 'jotai';

// 1. Define primitive atoms
export const countAtom = atom(0);
export const textAtom = atom('Hello Jotai');

export function Counter() {
  const [count, setCount] = useAtom(countAtom);

  return (
    <div>
      <h3>Count: {count}</h3>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
}
```

---

### Derived & Computed Atoms

Atoms can derive their value from other atoms:

```javascript
import { atom } from 'jotai';

export const priceAtom = atom(100);
export const taxRateAtom = atom(0.18);

// Read-only derived atom
export const totalPriceAtom = atom((get) => {
  const price = get(priceAtom);
  const tax = get(taxRateAtom);
  return price + price * tax;
});
```

---

### When to Choose Jotai

- Applications with high-frequency, granular state updates (e.g., drawing canvases, spreadsheet grids, nodes in a flowchart).
- Bottom-up state architectures where state nodes must be independently composed.

---

## Recoil (Brief Overview)

Recoil was Facebook's experimental atomic state management library. It pioneered the concept of atoms and selectors for React. However, as of recent years, development has slowed, and the React community has widely embraced **Jotai** and **Zustand** due to their active maintenance, lighter footprint, and compatibility with modern React releases.

---

## State Management Comparison Matrix

| Library | Paradigm | Bundle Size | Best For | Learning Curve |
|:---|:---|:---|:---|:---|
| **Context API** | Native React Context | 0 KB | Low-frequency data (Theme, Auth, Language) | Low |
| **Zustand** | Centralized Flux store | ~1.1 KB | General client state, dashboards, carts | Very Low |
| **Redux Toolkit** | Centralized Flux store | ~12 KB | Large enterprise codebases, complex flows | Moderate |
| **Jotai** | Atomic graph model | ~3 KB | Highly interconnected fine-grained state | Low |
| **TanStack Query** | Server state cache | ~13 KB | API fetching, caching, invalidation | Low-Moderate |

---

## Architecture & Selection Decision Guide

```
┌────────────────────────────────────────────────────────────────────────┐
│                        RECOMMENDED STACK (2026)                        │
├────────────────────────────────┬───────────────────────────────────────┤
│ Server Data / API Caching      │ TanStack Query (React Query)          │
│ Global Client Store (General)  │ Zustand                               │
│ Theme / Auth Session           │ Context API or Zustand                │
│ Complex Canvas / Grid State    │ Jotai                                 │
│ Large Enterprise Standard      │ Redux Toolkit                         │
└────────────────────────────────┴───────────────────────────────────────┘
```

---

## 🏋️ Practice Exercises

### Exercise 1: Shopping Cart with Zustand
Implement a store with:
- `items`: array of `{ id, title, price, quantity }`
- Actions: `addItem`, `decrementItem`, `removeItem`, `clearCart`
- Persisted to `localStorage` using the `persist` middleware.

### Exercise 2: Theme & Notification System with Context
Create two isolated contexts:
- `ThemeContext`: toggles `'light'` and `'dark'`.
- `ToastContext`: allows triggering temporary notification alerts across any page.

---

> **Previous:** [← 04 — Advanced Patterns](./04-advanced-patterns.md)  
> **Next:** [06 — Routing →](./06-routing.md)
