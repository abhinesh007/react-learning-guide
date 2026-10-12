# 📋 React Study Guide — Master Plan

> **Goal:** Create an extensive, self-contained React.js study guide covering basics → advanced → real-world project patterns. Each file is a standalone module with explanations, code examples, gotchas, and practice exercises.

---

## ✅ Status: All Modules Completed

| File | Status | Description |
|------|--------|-------------|
| `00-table-of-contents.md` | ✅ Done | Guide overview, prerequisite roadmap, setup |
| `01-fundamentals.md` | ✅ Done | JSX, Components, Props, Events, Lists, Strict Mode |
| `02-state-and-lifecycle.md` | ✅ Done | `useState`, Immutability, Lifecycle, `useEffect`, Traps |
| `03-hooks-deep-dive.md` | ✅ Done | All built-in hooks, React 19 hooks, Custom hooks |
| `04-advanced-patterns.md` | ✅ Done | HOCs, Render Props, Compound components, Portals, Errors |
| `05-state-management.md` | ✅ Done | Context API, Redux Toolkit, Zustand, Jotai, Matrix |
| `06-routing.md` | ✅ Done | React Router v6+, Loaders, Actions, Protected Routes |
| `07-forms.md` | ✅ Done | Controlled, Uncontrolled, React Hook Form, Zod, Uploads |
| `08-data-fetching.md` | ✅ Done | fetch, Axios, TanStack Query, SWR, API Architecture |
| `09-styling.md` | ✅ Done | CSS Modules, Tailwind CSS, Styled Components, Emotion |
| `10-testing.md` | ✅ Done | Vitest, React Testing Library, MSW, Playwright E2E |
| `11-performance-optimization.md` | ✅ Done | Profiler, memo, useMemo, useCallback, Virtualization |
| `12-typescript-with-react.md` | ✅ Done | Typing Props, Hooks, Events, Generics, Discriminated Unions |
| `13-real-world-patterns.md` | ✅ Done | Feature architecture, JWT auth, Error handling, Deployment |
| `14-ecosystem-and-tools.md` | ✅ Done | Libraries (Tables, Charts, Motion, UI kits, DnD, Utils) |

---


## 📝 Files To Create

---

### 📄 `02-state-and-lifecycle.md` — State & Lifecycle

**Sections:**
1. **What is State?** — State vs Props comparison table
2. **`useState` Hook**
   - Basic usage (counter example)
   - State with different types (string, number, boolean, object, array)
   - Updater function pattern (`setState(prev => ...)`) and why it matters
   - Lazy initialization (`useState(() => expensiveComputation())`)
3. **Updating Objects in State**
   - Why you must create new objects (immutability)
   - Spread operator for shallow updates
   - Nested object updates
   - Using `structuredClone()` for deep copies
4. **Updating Arrays in State**
   - Adding items (`[...arr, newItem]`)
   - Removing items (`.filter()`)
   - Updating items (`.map()`)
   - Sorting (copy first, then sort)
5. **Component Lifecycle (Conceptual)**
   - Mount → Update → Unmount diagram
   - How hooks map to lifecycle phases
6. **`useEffect` Hook**
   - What it is (synchronizing with external systems)
   - Dependency array explained (`[]`, `[dep]`, no array)
   - Cleanup function and why it matters
   - Common use cases: fetch data, event listeners, timers, DOM manipulation
7. **Common `useEffect` Mistakes**
   - Missing dependencies
   - Infinite loops
   - Race conditions in data fetching
   - Object/array dependencies (reference equality)
8. **`useLayoutEffect`** — When and why (DOM measurements before paint)
9. **Practice Exercises** — Timer, fetch user data, window resize tracker

---

### 📄 `03-hooks-deep-dive.md` — All Hooks Explained

**Sections:**
1. **Rules of Hooks**
   - Only call at the top level
   - Only call in React functions
   - ESLint plugin: `eslint-plugin-react-hooks`
2. **State Hooks**
   - `useState` (recap + advanced patterns)
   - `useReducer` — When state logic is complex
     - Reducer pattern, dispatch, action types
     - `useReducer` vs `useState` decision guide
3. **Context Hooks**
   - `useContext` — consuming context (full example)
4. **Ref Hooks**
   - `useRef` — DOM access + mutable values that don't trigger re-renders
   - `useImperativeHandle` — Customizing ref exposure
   - `forwardRef` — Forwarding refs to child components
5. **Effect Hooks**
   - `useEffect` (recap)
   - `useLayoutEffect` (recap)
   - `useInsertionEffect` (CSS-in-JS libraries)
6. **Performance Hooks**
   - `useMemo` — Memoize expensive computations
   - `useCallback` — Memoize functions
   - `useTransition` — Mark non-urgent state updates
   - `useDeferredValue` — Defer re-rendering of non-critical parts
7. **Other Hooks**
   - `useId` — Generate unique IDs for accessibility
   - `useSyncExternalStore` — Subscribe to external stores
   - `useDebugValue` — Label custom hooks in DevTools
8. **React 19 Hooks (New)**
   - `useActionState` — Form action state management
   - `useFormStatus` — Pending state for forms
   - `useOptimistic` — Optimistic UI updates
   - `use` — Read resources in render (promises, context)
9. **Custom Hooks**
   - What they are and naming convention (`use` prefix)
   - Examples: `useLocalStorage`, `useDebounce`, `useFetch`, `useMediaQuery`, `useOnClickOutside`
   - Composing hooks together
   - Testing custom hooks
10. **Practice Exercises** — Build `useToggle`, `usePrevious`, `useWindowSize`

---

### 📄 `04-advanced-patterns.md` — Advanced Component Patterns

**Sections:**
1. **Higher-Order Components (HOCs)**
   - Pattern explanation with diagram
   - Building a `withAuth` HOC
   - Building a `withLoading` HOC
   - Pros, cons, and when to use
2. **Render Props**
   - Pattern explanation
   - Mouse tracker example
   - Render props vs hooks comparison
3. **Compound Components**
   - Pattern with `React.Children` and Context
   - Building a `<Tabs>` / `<Tab>` compound component
   - Building an `<Accordion>` compound component
4. **Controlled vs Uncontrolled Components**
   - When to use each
   - Building a controlled `<Modal>`
5. **Portals**
   - `createPortal` for modals, tooltips, toasts
   - Event bubbling through portals
6. **Error Boundaries**
   - Class-based error boundary (only way currently)
   - `getDerivedStateFromError` + `componentDidCatch`
   - Reusable `<ErrorBoundary>` component
   - Using `react-error-boundary` library
7. **Suspense & Lazy Loading**
   - `React.lazy()` for code splitting
   - `<Suspense>` fallback UI
   - Route-based code splitting
8. **Context Module Pattern** — Extracting context logic into a separate module
9. **State Initializer Pattern** — Reset component to initial state
10. **Practice Exercises** — Build a compound `<Select>/<Option>` component

---

### 📄 `05-state-management.md` — State Management

**Sections:**
1. **When Do You Need State Management?**
   - Prop drilling problem explained
   - Local vs global state decision tree
   - State categories: UI state, server state, URL state, form state
2. **Context API (Built-in)**
   - `createContext`, `Provider`, `useContext`
   - Theme switcher full example
   - Auth context full example
   - Limitations: performance (re-renders), not ideal for frequent updates
   - Optimization: splitting contexts, memoizing value
3. **Redux Toolkit (Industry Standard)**
   - Core concepts: Store, Slices, Actions, Reducers
   - Setting up Redux Toolkit (`configureStore`, `createSlice`)
   - `useSelector` and `useDispatch`
   - Async logic with `createAsyncThunk`
   - RTK Query for data fetching
   - Redux DevTools
   - When to use Redux (large teams, complex shared state)
4. **Zustand (Lightweight & Trending)**
   - Why Zustand is gaining popularity
   - Creating a store
   - Selectors and performance
   - Middleware: `persist`, `devtools`, `immer`
   - Zustand vs Redux comparison
5. **Jotai (Atomic State)**
   - Atom-based model
   - `atom`, `useAtom`
   - Derived atoms
   - When to choose Jotai
6. **Recoil (Facebook's Experiment)**
   - Atoms and Selectors
   - Brief overview (less popular now)
7. **Comparison Table** — Redux vs Zustand vs Jotai vs Context
8. **Recommendation Guide** — Which to pick for which project type
9. **Practice Exercises** — Shopping cart with Zustand, Theme system with Context

---

### 📄 `06-routing.md` — Routing with React Router

**Sections:**
1. **Why Client-Side Routing?** — SPA vs MPA, how it works
2. **React Router v6+ Setup**
   - Installation
   - `BrowserRouter`, `Routes`, `Route`
   - `<Link>` and `<NavLink>`
3. **Dynamic Routes** — Route params (`:id`), `useParams()`
4. **Nested Routes & Layouts**
   - `<Outlet>` for nested content
   - Shared layout pattern
5. **Programmatic Navigation**
   - `useNavigate()` hook
   - Navigate with state
   - Replace vs push
6. **Search Params** — `useSearchParams()` for query strings
7. **Protected/Private Routes**
   - Auth guard pattern
   - Role-based routing
   - Redirect after login
8. **Lazy Loading Routes**
   - `React.lazy()` + `Suspense` per route
   - Loading indicators
9. **404 / Not Found Route** — Catch-all routes
10. **Data Loading (Loaders & Actions)** — React Router v6.4+ data APIs
11. **Route Configuration Object** — `createBrowserRouter` + `createRoutesFromElements`
12. **`useLocation`**, **`useMatch`** hooks
13. **Practice Exercises** — Multi-page app with auth, nested dashboard routes

---

### 📄 `07-forms.md` — Forms in React

**Sections:**
1. **Controlled Components**
   - Input, textarea, select, checkbox, radio
   - Handling multiple inputs with a single handler
   - Controlled form full example
2. **Uncontrolled Components**
   - `useRef` for form values
   - `defaultValue` / `defaultChecked`
   - When to use uncontrolled (file inputs, third-party DOM libs)
3. **Form Validation (Manual)**
   - Real-time vs on-submit validation
   - Error state management
   - Accessible error messages
4. **React Hook Form (Most Popular)**
   - Installation & setup
   - `useForm`, `register`, `handleSubmit`
   - Validation rules (required, min, max, pattern, custom)
   - Error messages
   - Watch, reset, setValue
   - Form with multiple steps (wizard)
   - Performance advantages (uncontrolled under the hood)
5. **Zod + React Hook Form**
   - Schema-based validation with Zod
   - `zodResolver` integration
   - Type-safe forms with TypeScript
6. **Yup + Formik (Alternative)**
   - Formik basics: `useFormik`, `<Formik>`, `<Field>`, `<ErrorMessage>`
   - Yup schema validation
   - When to choose Formik vs React Hook Form
7. **File Uploads** — Handling file inputs, preview, drag & drop basics
8. **Comparison Table** — React Hook Form vs Formik vs Native
9. **Practice Exercises** — Registration form with RHF + Zod, multi-step checkout form

---

### 📄 `08-data-fetching.md` — Data Fetching

**Sections:**
1. **`fetch` API Basics**
   - GET, POST, PUT, DELETE
   - Handling loading, error, and data states
   - AbortController for cleanup
2. **Axios**
   - Installation and why use it over fetch
   - Interceptors (auth tokens, error handling)
   - Creating an Axios instance with base URL
   - Request/response transformation
3. **Custom `useFetch` Hook** — Building a reusable data fetching hook
4. **TanStack Query (React Query) — The Standard**
   - Why TanStack Query? (caching, deduplication, background refetch, pagination)
   - `QueryClient`, `QueryClientProvider`
   - `useQuery` — Fetching data
   - `useMutation` — Creating/updating/deleting data
   - Query keys and cache invalidation
   - Pagination and infinite scroll (`useInfiniteQuery`)
   - Optimistic updates
   - Prefetching
   - DevTools
5. **SWR (Alternative by Vercel)**
   - `useSWR` basics
   - SWR vs TanStack Query comparison
6. **Suspense for Data Fetching** — Future/experimental patterns
7. **API Layer Architecture** — Organizing API calls in a real project
8. **Practice Exercises** — CRUD app with TanStack Query, infinite scroll feed

---

### 📄 `09-styling.md` — Styling in React

**Sections:**
1. **Inline Styles** — Object syntax, dynamic styles, limitations
2. **Plain CSS & CSS Files** — Importing `.css` files, global styles
3. **CSS Modules**
   - `.module.css` naming convention
   - Scoped class names
   - Composing styles
   - Using with Vite
4. **CSS-in-JS: Styled Components**
   - Tagged template literals
   - Dynamic styles based on props
   - Theming with `<ThemeProvider>`
   - Global styles
   - When to use / not use
5. **CSS-in-JS: Emotion**
   - `css` prop and `styled` API
   - Brief comparison with Styled Components
6. **Tailwind CSS with React**
   - Setup with Vite
   - Utility-first approach
   - Conditional classes with `clsx` / `cn`
   - `tailwind-merge` for conflict resolution
   - Component patterns with Tailwind
7. **Comparison Table** — Modules vs Styled Components vs Tailwind
8. **Best Practices** — Design tokens, responsive design, dark mode
9. **Practice Exercises** — Theme switcher with CSS variables, component library card

---

### 📄 `10-testing.md` — Testing React Applications

**Sections:**
1. **Testing Philosophy**
   - Testing pyramid (unit → integration → e2e)
   - What to test in React
   - Testing user behavior, not implementation
2. **Setup: Vitest + React Testing Library**
   - Installation and configuration
   - `jsdom` environment
   - Setup file for cleanup
3. **React Testing Library (RTL)**
   - `render`, `screen`, `fireEvent`, `waitFor`
   - Query priority: `getByRole` > `getByLabelText` > `getByText` > `getByTestId`
   - `userEvent` vs `fireEvent`
   - Testing async components
4. **Common Test Patterns**
   - Testing component renders
   - Testing user interactions
   - Testing conditional rendering
   - Testing forms
   - Testing API calls (mocking)
   - Testing custom hooks (`renderHook`)
5. **Mocking**
   - Mocking modules (`vi.mock`)
   - Mocking API calls with MSW (Mock Service Worker)
   - Mocking React Router
   - Mocking context providers
6. **Snapshot Testing** — When useful, when harmful
7. **E2E Testing with Playwright**
   - Brief overview
   - Setup with React/Vite
   - Writing a basic E2E test
8. **Code Coverage** — Configuration and meaningful metrics
9. **Practice Exercises** — Test a login form, test a todo list CRUD

---

### 📄 `11-performance-optimization.md` — Performance

**Sections:**
1. **How React Renders** — Trigger → Render → Commit
2. **When Does a Component Re-render?**
   - State change, parent re-render, context change
   - Re-render ≠ DOM update (diffing still applies)
3. **React DevTools Profiler** — Finding performance bottlenecks
4. **`React.memo`** — Skip re-renders for unchanged props
   - Default shallow comparison
   - Custom comparison function
   - When to use / not use
5. **`useMemo`** — Memoize expensive computed values
   - When it helps vs when it's premature
   - Examples: filtering large lists, complex calculations
6. **`useCallback`** — Memoize callback functions
   - Why functions cause re-renders in children
   - Combining `useCallback` with `React.memo`
7. **`useTransition` & `useDeferredValue`** — Concurrent features
   - Keeping UI responsive during heavy updates
   - Search input with deferred results
8. **Code Splitting**
   - `React.lazy` + `Suspense`
   - Route-level splitting
   - Component-level splitting
9. **Virtualization** — Rendering huge lists
   - `@tanstack/react-virtual`
   - `react-window` / `react-virtuoso`
10. **Image Optimization** — Lazy loading images, `loading="lazy"`
11. **Debouncing & Throttling** — Search inputs, scroll handlers
12. **Bundle Analysis** — `rollup-plugin-visualizer`, tree shaking
13. **Performance Checklist** — Quick reference for auditing
14. **Practice Exercises** — Optimize a slow dashboard, virtualize a 10k-item list

---

### 📄 `12-typescript-with-react.md` — TypeScript + React

**Sections:**
1. **Why TypeScript with React?** — Benefits, adoption stats
2. **Project Setup** — Vite + React + TypeScript
3. **Typing Components**
   - Props interfaces vs types
   - Optional props, default values
   - Children type (`React.ReactNode`, `React.PropsWithChildren`)
4. **Typing Hooks**
   - `useState<Type>()` 
   - `useRef<HTMLInputElement>(null)`
   - `useReducer` with discriminated unions
   - `useContext` with proper typing
5. **Typing Events**
   - `React.ChangeEvent<HTMLInputElement>`
   - `React.FormEvent<HTMLFormElement>`
   - `React.MouseEvent<HTMLButtonElement>`
   - Common event types reference table
6. **Typing Styles** — `React.CSSProperties`
7. **Generics in Components** — Reusable typed components (e.g., `<List<T>>`)
8. **Utility Types for React**
   - `React.ComponentProps<typeof Button>`
   - `React.HTMLAttributes<HTMLDivElement>`
   - `Omit`, `Pick`, `Partial`, `Required`
9. **Discriminated Unions** — For variant/status props
10. **Type-Safe API Responses** — Typing fetch/axios responses
11. **Common Patterns**
    - Polymorphic `as` prop
    - Exhaustive switch with `never`
    - Template literal types for variants
12. **Practice Exercises** — Convert a JS React app to TypeScript

---

### 📄 `13-real-world-patterns.md` — Real-World Project Patterns

**Sections:**
1. **Project Structure**
   - Feature-based (recommended) vs type-based
   - Detailed folder structure example
   - Barrel exports (`index.ts`)
2. **Environment Configuration**
   - `.env` files with Vite (`VITE_` prefix)
   - Runtime vs build-time config
3. **API Layer Architecture**
   - Axios instance with interceptors
   - Service modules per domain (authService, userService)
   - Error handling middleware
   - Token refresh pattern
4. **Authentication Patterns**
   - JWT flow (access + refresh tokens)
   - Auth context + protected routes
   - Persisting auth state
   - Auto-logout on token expiry
5. **Global Error Handling**
   - Error boundaries per section
   - Toast notifications for API errors
   - Fallback UIs
6. **Loading States & Skeletons** — Skeleton screens, shimmer effects
7. **Toast / Notification System** — `react-hot-toast`, `sonner`
8. **Modal Management** — Portal-based modals, confirmation dialogs
9. **Dark Mode Implementation** — CSS variables, system preference detection
10. **Internationalization (i18n)** — `react-i18next` basics
11. **SEO in SPAs** — `react-helmet-async`, meta tags
12. **Deployment** — Vercel, Netlify, Docker basics
13. **Code Quality** — ESLint, Prettier, Husky pre-commit hooks
14. **Practice Exercises** — Set up a production-ready project skeleton

---

### 📄 `14-ecosystem-and-tools.md` — Ecosystem & Essential Libraries

**Sections:**
1. **Build Tools**
   - Vite (recommended, why it's fast)
   - Create React App (legacy, avoid)
   - Next.js (when you need SSR/SSG)
   - Remix (alternative full-stack)
2. **UI Component Libraries**
   - shadcn/ui (copy-paste, customizable, Tailwind-based)
   - Material UI (MUI) — enterprise, Google Material Design
   - Ant Design — enterprise, feature-rich
   - Chakra UI — accessible, DX-focused
   - Mantine — batteries-included
   - Comparison table with use cases
3. **Animation Libraries**
   - Framer Motion — most popular, declarative
   - React Spring — physics-based
   - Auto-animate — zero-config animations
   - GSAP with React
4. **Date/Time**
   - `date-fns` (functional, tree-shakeable)
   - `dayjs` (lightweight Moment replacement)
5. **HTTP / Data**
   - Axios
   - TanStack Query
   - SWR
6. **Tables** — TanStack Table (headless, powerful)
7. **Charts** — Recharts, Chart.js with react-chartjs-2, Nivo
8. **Drag & Drop** — `@dnd-kit`, `react-beautiful-dnd` (deprecated)
9. **Rich Text Editors** — TipTap, Slate, Lexical (Meta)
10. **PDF Generation** — `@react-pdf/renderer`
11. **Icons** — `react-icons`, `lucide-react`
12. **Utilities**
    - `clsx` / `classnames` — Conditional class names
    - `nanoid` / `uuid` — ID generation
    - `lodash` (use specific imports for tree-shaking)
    - `immer` — Immutable state updates
    - `zod` — Schema validation
13. **DevTools** — React DevTools, Redux DevTools, TanStack Query DevTools
14. **Must-Know NPM Commands** — Quick reference

---

## 📐 Standards for Each File

Every file will follow this consistent structure:

```
# Title

> One-line summary

---

## Table of Contents (linked)

## Section 1
  - Explanation with real-world context
  - Code examples (copy-paste ready)
  - ⚠️ Common gotchas / mistakes
  - ✅ Best practices

## Section N
  ...

## 🏋️ Practice Exercises
  - Hands-on tasks to reinforce concepts

---

> **Next:** [Link to next module →]
> **Previous:** [← Link to previous module]
```

**Formatting conventions:**
- ✅ for best practices / correct patterns
- ❌ for anti-patterns / common mistakes
- ⚠️ for gotchas and warnings
- 💡 for tips and insights
- Tables for comparisons
- Diagrams (ASCII) for flows and architecture

---

## 📊 Estimated Output

| Metric | Estimate |
|--------|----------|
| Total files | 15 (including TOC + plan) |
| Total sections | ~130+ |
| Code examples | ~200+ |
| Practice exercises | ~30+ |
| Estimated reading time | 4-6 hours (full guide) |

---

## 🚀 Execution Order

1. ~~`00-table-of-contents.md`~~ ✅
2. ~~`01-fundamentals.md`~~ ✅
3. ~~`02-state-and-lifecycle.md`~~ ✅
4. ~~`03-hooks-deep-dive.md`~~ ✅
5. ~~`04-advanced-patterns.md`~~ ✅
6. ~~`05-state-management.md`~~ ✅
7. ~~`06-routing.md`~~ ✅
8. ~~`07-forms.md`~~ ✅
9. ~~`08-data-fetching.md`~~ ✅
10. ~~`09-styling.md`~~ ✅
11. ~~`10-testing.md`~~ ✅
12. ~~`11-performance-optimization.md`~~ ✅
13. ~~`12-typescript-with-react.md`~~ ✅
14. ~~`13-real-world-patterns.md`~~ ✅
15. ~~`14-ecosystem-and-tools.md`~~ ✅

---

> 🎉 **All 15 modules have been generated!** The complete comprehensive study guide is available in this folder.

