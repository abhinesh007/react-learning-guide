# 06 — Routing with React Router

> Everything you need to architect client-side navigation in modern SPAs: React Router v6+, nested layouts, dynamic params, query strings, loaders, and protected auth guards.

---

## Table of Contents

- [Client-Side Routing Explained](#client-side-routing-explained)
  - [SPA vs. MPA Architecture](#spa-vs-mpa-architecture)
  - [How the Browser History API Works](#how-the-browser-history-api-works)
- [React Router v6+ Core Setup](#react-router-v6-core-setup)
  - [Installation](#installation)
  - [Declarative Syntax: `<BrowserRouter>`, `<Routes>`, `<Route>`](#declarative-syntax-browserrouter-routes-route)
  - [Navigation Links: `<Link>` vs `<NavLink>`](#navigation-links-link-vs-navlink)
- [Dynamic Route Segments](#dynamic-route-segments)
  - [URL Parameters (`:id`) & `useParams`](#url-parameters-id--useparams)
- [Nested Routes & Layout Hierarchies](#nested-routes--layout-hierarchies)
  - [The `<Outlet>` Component](#the-outlet-component)
  - [Shared Layout Architecture](#shared-layout-architecture)
  - [Index Routes](#index-routes)
- [Programmatic Navigation](#programmatic-navigation)
  - [The `useNavigate` Hook](#the-usenavigate-hook)
  - [Push vs. Replace Navigation](#push-vs-replace-navigation)
  - [Passing Ephemeral State Across Routes](#passing-ephemeral-state-across-routes)
- [URL Search Parameters (Query Strings)](#url-search-parameters-query-strings)
  - [Reading and Writing with `useSearchParams`](#reading-and-writing-with-usesearchparams)
- [Protected & Private Routes](#protected--private-routes)
  - [Authentication Guards](#authentication-guards)
  - [Role-Based Access Control (RBAC)](#role-based-access-control-rbac)
  - [Redirect Back After Login Pattern](#redirect-back-after-login-pattern)
- [Route-Based Code Splitting](#route-based-code-splitting)
  - [`React.lazy()` + `<Suspense>` per Route](#reactlazy--suspense-per-route)
- [404 Not Found & Catch-All Routes](#404-not-found--catch-all-routes)
- [Modern Data APIs (v6.4+)](#modern-data-apis-v64)
  - [`createBrowserRouter` & `RouterProvider`](#createbrowserrouter--routerprovider)
  - [Data `loader` Functions](#data-loader-functions)
  - [Data `action` Functions](#data-action-functions)
  - [`useNavigation` & Global Loading Spinners](#usenavigation--global-loading-spinners)
- [Utility Hooks: `useLocation` & `useMatch`](#utility-hooks-uselocation--usematch)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## Client-Side Routing Explained

### SPA vs. MPA Architecture

In a traditional **Multi-Page Application (MPA)**, every navigation link triggers a full page refresh: the browser makes an HTTP request to the server, waits for an HTML document, tears down the current DOM, and paints the new document.

In a **Single-Page Application (SPA)** with client-side routing:
1. The initial HTML, JavaScript, and CSS bundle loads once.
2. When a user clicks a link, JavaScript intercepts the click and prevents the browser's default full page reload.
3. React Router updates the browser address bar using the HTML5 `history.pushState()` API.
4. React swaps out the rendered components matching the new URL in milliseconds without flashing a blank screen.

```
Traditional MPA:
Click link ──► Request server ──► Server renders HTML ──► Full page reload (Blank flash)

React SPA (Client-Side Routing):
Click link ──► Intercept event ──► history.pushState() ──► Swap Component in DOM (Instant)
```

---

## React Router v6+ Core Setup

### Installation

```bash
npm install react-router-dom
```

---

### Declarative Syntax: `<BrowserRouter>`, `<Routes>`, `<Route>`

```jsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import Home from './pages/Home';
import About from './pages/About';
import Contact from './pages/Contact';

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/contact" element={<Contact />} />
      </Routes>
    </BrowserRouter>
  );
}
```

---

### Navigation Links: `<Link>` vs `<NavLink>`

- `<Link>`: Renders an anchor tag `<a>` that navigates client-side without page reload.
- `<NavLink>`: Special version of `<Link>` that knows whether it is "active" (useful for navigation bars).

```jsx
import { Link, NavLink } from 'react-router-dom';

export function NavigationBar() {
  return (
    <nav className="navbar">
      {/* Plain Link */}
      <Link to="/" className="brand-logo">Acme Inc</Link>

      {/* NavLink with automatic active class */}
      <NavLink
        to="/dashboard"
        className={({ isActive }) => (isActive ? 'nav-link active' : 'nav-link')}
      >
        Dashboard
      </NavLink>

      {/* NavLink with dynamic inline styles */}
      <NavLink
        to="/profile"
        style={({ isActive }) => ({
          color: isActive ? '#3b82f6' : '#6b7280',
          fontWeight: isActive ? 'bold' : 'normal',
        })}
      >
        Profile
      </NavLink>
    </nav>
  );
}
```

---

## Dynamic Route Segments

### URL Parameters (`:id`) & `useParams`

Dynamic segments match variable path segments (e.g., user IDs, post slugs, product SKUs):

```jsx
// Route Definition
<Route path="/products/:productId" element={<ProductDetail />} />
```

```jsx
// src/pages/ProductDetail.jsx
import { useParams, Link } from 'react-router-dom';
import { useState, useEffect } from 'react';

export default function ProductDetail() {
  // Extract route parameters defined with :productId
  const { productId } = useParams();
  const [product, setProduct] = useState(null);

  useEffect(() => {
    fetch(`/api/products/${productId}`)
      .then(res => res.json())
      .then(data => setProduct(data));
  }, [productId]);

  if (!product) return <div>Loading product #{productId}...</div>;

  return (
    <div>
      <Link to="/products">← Back to all products</Link>
      <h1>{product.name}</h1>
      <p>Price: ${product.price}</p>
    </div>
  );
}
```

---

## Nested Routes & Layout Hierarchies

### The `<Outlet>` Component

Nested routing lets you render sub-routes inside parent layouts. The parent component defines layout structure (header, sidebar, footer) and places an `<Outlet />` placeholder where child routes are rendered.

```
┌────────────────────────────────────────────────────────┐
│ AppLayout                                              │
│ ┌───────────────┐ ┌──────────────────────────────────┐ │
│ │ Sidebar Nav   │ │ <Outlet />                       │ │
│ │ - Overview    │ │ (Renders DashboardOverview,      │ │
│ │ - Analytics   │ │  AnalyticsView, or SettingsView  │ │
│ │ - Settings    │ │  based on current URL)           │ │
│ └───────────────┘ └──────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

---

### Shared Layout Architecture

```jsx
import { Routes, Route, Outlet, Link } from 'react-router-dom';

// 1. Shared Layout Shell
function DashboardLayout() {
  return (
    <div className="dashboard-grid">
      <aside className="sidebar">
        <h3>Portal Admin</h3>
        <nav>
          <Link to="/dashboard">Overview</Link>
          <Link to="/dashboard/analytics">Analytics</Link>
          <Link to="/dashboard/settings">Settings</Link>
        </nav>
      </aside>

      <main className="content-pane">
        {/* Child routes render here */}
        <Outlet />
      </main>
    </div>
  );
}

// 2. Child Page Components
function DashboardHome() {
  return <h2>Welcome to your Dashboard Overview</h2>;
}

function Analytics() {
  return <h2>Performance & Metrics</h2>;
}

function Settings() {
  return <h2>Account Settings</h2>;
}

// 3. Nested Route Tree
export default function App() {
  return (
    <Routes>
      <Route path="/dashboard" element={<DashboardLayout />}>
        {/* Index route: renders when URL matches exact parent path "/dashboard" */}
        <Route index element={<DashboardHome />} />
        <Route path="analytics" element={<Analytics />} />
        <Route path="settings" element={<Settings />} />
      </Route>
    </Routes>
  );
}
```

---

## Programmatic Navigation

### The `useNavigate` Hook

Use `useNavigate` when navigation must occur programmatically in response to an event (e.g., after form submission, token expiration, button click):

```jsx
import { useNavigate } from 'react-router-dom';

export function LoginForm() {
  const navigate = useNavigate();

  const handleLogin = async (e) => {
    e.preventDefault();
    const success = await authenticateUser();
    
    if (success) {
      // Navigate to dashboard
      navigate('/dashboard');
    }
  };

  return <form onSubmit={handleLogin}>...</form>;
}
```

---

### Push vs. Replace Navigation

- `navigate('/path')`: **Pushes** a new entry onto the browser's history stack. The user can press the browser "Back" button to return.
- `navigate('/path', { replace: true })`: **Replaces** the current history entry. Useful for login redirects so hitting "Back" doesn't return to the login screen.
- `navigate(-1)`: Goes back one step in history (like `window.history.back()`).

---

### Passing Ephemeral State Across Routes

You can attach transient state to a navigation transition that doesn't appear in the URL:

```jsx
// Sending component:
navigate('/checkout/confirmation', {
  state: { orderId: 'ORD-98234', total: 149.99 }
});

// Receiving component:
import { useLocation } from 'react-router-dom';

export function OrderConfirmation() {
  const location = useLocation();
  const orderDetails = location.state; // { orderId: 'ORD-98234', total: 149.99 }

  if (!orderDetails) {
    return <p>No active order found.</p>;
  }

  return <div>Order #{orderDetails.orderId} processed successfully!</div>;
}
```

---

## URL Search Parameters (Query Strings)

Search parameters (e.g., `?category=laptops&sort=price_asc&page=2`) allow URL shareability and browser bookmarking.

### Reading and Writing with `useSearchParams`

```jsx
import { useSearchParams } from 'react-router-dom';

export function ProductCatalog() {
  const [searchParams, setSearchParams] = useSearchParams();

  // Read search parameters with defaults
  const query = searchParams.get('q') || '';
  const sort = searchParams.get('sort') || 'featured';
  const page = parseInt(searchParams.get('page') || '1', 10);

  const handleFilterChange = (newSort) => {
    // Preserves existing params while updating specific fields
    setSearchParams(prev => {
      prev.set('sort', newSort);
      prev.set('page', '1'); // Reset to page 1 on filter
      return prev;
    });
  };

  return (
    <div>
      <input
        type="search"
        value={query}
        placeholder="Filter products..."
        onChange={(e) => {
          setSearchParams(prev => {
            if (e.target.value) {
              prev.set('q', e.target.value);
            } else {
              prev.delete('q');
            }
            return prev;
          });
        }}
      />

      <select value={sort} onChange={(e) => handleFilterChange(e.target.value)}>
        <option value="featured">Featured</option>
        <option value="price_asc">Price: Low to High</option>
        <option value="price_desc">Price: High to Low</option>
      </select>
    </div>
  );
}
```

---

## Protected & Private Routes

### Authentication Guards

```jsx
import { Navigate, Outlet, useLocation } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export function ProtectedRoute() {
  const { isAuthenticated, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) {
    return <div>Verifying session...</div>;
  }

  if (!isAuthenticated) {
    // Redirect to /login, remembering the attempted URL in state
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  // Render the requested protected child route
  return <Outlet />;
}
```

---

### Role-Based Access Control (RBAC)

```jsx
export function RoleGuard({ allowedRoles }) {
  const { user } = useAuth();

  if (!user || !allowedRoles.includes(user.role)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return <Outlet />;
}

// Route Tree Usage:
<Routes>
  {/* Public */}
  <Route path="/login" element={<LoginPage />} />

  {/* Authenticated Members */}
  <Route element={<ProtectedRoute />}>
    <Route path="/dashboard" element={<Dashboard />} />
    
    {/* Admin Only */}
    <Route element={<RoleGuard allowedRoles={['admin', 'superadmin']} />}>
      <Route path="/admin/users" element={<UserAdmin />} />
      <Route path="/admin/billing" element={<BillingAdmin />} />
    </Route>
  </Route>
</Routes>
```

---

### Redirect Back After Login Pattern

```jsx
// src/pages/LoginPage.jsx
import { useLocation, useNavigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export default function LoginPage() {
  const { login } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();

  // Retrieve origin destination, default to /dashboard
  const destination = location.state?.from?.pathname || '/dashboard';

  const handleSubmit = async (e) => {
    e.preventDefault();
    await login({ username, password });
    // Navigate user back to the exact page they originally tried to visit!
    navigate(destination, { replace: true });
  };

  return <form onSubmit={handleSubmit}>...</form>;
}
```

---

## Route-Based Code Splitting

Load route bundles on demand using `React.lazy()` wrapped in a `<Suspense>` boundary:

```jsx
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

// Lazy page imports
const Home = lazy(() => import('./pages/Home'));
const Analytics = lazy(() => import('./pages/Analytics'));
const Settings = lazy(() => import('./pages/Settings'));

export default function App() {
  return (
    <Suspense fallback={<div className="page-spinner">Loading page...</div>}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/analytics" element={<Analytics />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

---

## 404 Not Found & Catch-All Routes

Use the wildcard asterisk `path="*"` route at the very bottom of your `<Routes>` configuration:

```jsx
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/about" element={<About />} />
  
  {/* Catch-all 404 route */}
  <Route path="*" element={<NotFoundPage />} />
</Routes>
```

```jsx
export function NotFoundPage() {
  return (
    <div style={{ textAlign: 'center', padding: '60px' }}>
      <h1>404 — Page Not Found</h1>
      <p>The page you are looking for does not exist.</p>
      <Link to="/" className="btn-primary">Return Home</Link>
    </div>
  );
}
```

---

## Modern Data APIs (v6.4+)

React Router v6.4 introduced data loaders, actions, and fetchers, fetching data in parallel *before* the component even begins rendering.

### `createBrowserRouter` & `RouterProvider`

```jsx
import {
  createBrowserRouter,
  RouterProvider,
  useLoaderData,
} from 'react-router-dom';

// 1. Data Loader Function
async function userLoader({ params }) {
  const res = await fetch(`/api/users/${params.userId}`);
  if (!res.ok) throw new Response('User Not Found', { status: 404 });
  return res.json();
}

// 2. Component Consuming Loader Data
function UserProfile() {
  const user = useLoaderData(); // Pre-fetched data available immediately!
  return <h1>{user.name} ({user.email})</h1>;
}

// 3. Router Configuration Object
const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      {
        path: 'users/:userId',
        element: <UserProfile />,
        loader: userLoader,
      },
    ],
  },
]);

export default function App() {
  return <RouterProvider router={router} />;
}
```

---

### Data `action` Functions

Actions handle data mutations (POST, PUT, DELETE) triggered by React Router's `<Form>`:

```jsx
import { Form, redirect } from 'react-router-dom';

export async function newPostAction({ request }) {
  const formData = await request.formData();
  const title = formData.get('title');
  const body = formData.get('body');

  await fetch('/api/posts', {
    method: 'POST',
    body: JSON.stringify({ title, body }),
  });

  return redirect('/posts'); // Automatically revalidates loaders & redirects!
}

export function NewPost() {
  return (
    <Form method="post">
      <input name="title" required />
      <textarea name="body" required />
      <button type="submit">Create Post</button>
    </Form>
  );
}
```

---

## Utility Hooks: `useLocation` & `useMatch`

### `useLocation`
Returns the current `location` object representing the current URL (pathname, search, hash, state). Excellent for tracking page analytics or route changes:

```jsx
import { useLocation } from 'react-router-dom';
import { useEffect } from 'react';

export function PageAnalyticsTracker() {
  const location = useLocation();

  useEffect(() => {
    window.gtag?.('send', 'pageview', location.pathname);
  }, [location]);

  return null;
}
```

### `useMatch`
Tests if a given path pattern matches the active route:

```jsx
import { useMatch } from 'react-router-dom';

export function CustomTab({ to, label }) {
  const match = useMatch(to);
  return (
    <div className={`tab ${match ? 'active-tab' : ''}`}>
      {label}
    </div>
  );
}
```

---

## 🏋️ Practice Exercises

### Exercise 1: Multi-Page App with Auth & Role Gates
Build an application tree with:
- `/login` (public)
- `/dashboard` (protected, requires login)
- `/dashboard/team` (protected, admin-only)
- Redirect to `/login` with `from` state; return to target upon successful authentication.

### Exercise 2: Search Catalog with Query String Sync
Create a product directory where:
- Search text, category dropdown, and page number are read from and written to `useSearchParams`.
- Refreshing the browser or sharing the URL preserves all active filters.

---

> **Previous:** [← 05 — State Management](./05-state-management.md)  
> **Next:** [07 — Forms →](./07-forms.md)
