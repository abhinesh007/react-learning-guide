# 08 — Data Fetching in React

> Master asynchronous data handling in React: native `fetch` with AbortController, Axios with request interceptors, TanStack Query (React Query), SWR, and clean API architecture.

---

## Table of Contents

- [The Native `fetch` API](#the-native-fetch-api)
  - [Managing Loading, Error, and Data States](#managing-loading-error-and-data-states)
  - [Canceling Requests with `AbortController`](#canceling-requests-with-abortcontroller)
- [Axios in Production](#axios-in-production)
  - [Why Axios over Native `fetch`?](#why-axios-over-native-fetch)
  - [Configuring a Centralized Axios Instance](#configuring-a-centralized-axios-instance)
  - [Interceptors for Auth Tokens & Token Refresh](#interceptors-for-auth-tokens--token-refresh)
- [Building a Reusable `useFetch` Custom Hook](#building-a-reusable-usefetch-custom-hook)
- [TanStack Query (React Query) — The Gold Standard](#tanstack-query-react-query--the-gold-standard)
  - [Why Server State Libraries are Essential](#why-server-state-libraries-are-essential)
  - [Setup: `QueryClient` & `QueryClientProvider`](#setup-queryclient--queryclientprovider)
  - [Fetching with `useQuery`](#fetching-with-usequery)
  - [Mutations with `useMutation`](#mutations-with-usemutation)
  - [Query Keys & Cache Invalidation](#query-keys--cache-invalidation)
  - [Pagination & Infinite Scrolling (`useInfiniteQuery`)](#pagination--infinite-scrolling-useinfinitequery)
  - [Optimistic Updates Pattern](#optimistic-updates-pattern)
  - [Query Prefetching & React Query DevTools](#query-prefetching--react-query-devtools)
- [SWR (Alternative by Vercel)](#swr-alternative-by-vercel)
  - [`useSWR` Stale-While-Revalidate Basics](#useswr-stale-while-revalidate-basics)
  - [SWR vs. TanStack Query Comparison](#swr-vs-tanstack-query-comparison)
- [Suspense for Data Fetching](#suspense-for-data-fetching)
- [Enterprise API Layer Architecture](#enterprise-api-layer-architecture)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## The Native `fetch` API

### Managing Loading, Error, and Data States

When fetching data manually in `useEffect`, always handle the three canonical states: **Loading**, **Error**, and **Success (Data)**:

```jsx
import { useState, useEffect } from 'react';

export function UserList() {
  const [users, setUsers] = useState([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function loadData() {
      try {
        setIsLoading(true);
        setError(null);

        const response = await fetch('https://jsonplaceholder.typicode.com/users');
        
        // ⚠️ fetch() does NOT reject on HTTP 404 or 500! Must check response.ok manually:
        if (!response.ok) {
          throw new Error(`HTTP Error! Status: ${response.status}`);
        }

        const data = await response.json();
        setUsers(data);
      } catch (err) {
        setError(err.message || 'An unexpected error occurred');
      } finally {
        setIsLoading(false);
      }
    }

    loadData();
  }, []);

  if (isLoading) return <div>Loading user list...</div>;
  if (error) return <div className="error-box">Error: {error}</div>;

  return (
    <ul>
      {users.map(u => (
        <li key={u.id}>{u.name} — {u.email}</li>
      ))}
    </ul>
  );
}
```

---

### Canceling Requests with `AbortController`

If a user rapidly navigates between tabs or types in an autocomplete search box, pending HTTP requests must be aborted to prevent race conditions and wasted bandwidth:

```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/search?q=${query}`, { signal: controller.signal })
    .then(res => res.json())
    .then(data => setResults(data))
    .catch(err => {
      // Abort is not an error! Ignore AbortError:
      if (err.name !== 'AbortError') {
        setError(err.message);
      }
    });

  return () => {
    // Abort HTTP request if query changes or component unmounts
    controller.abort();
  };
}, [query]);
```

---

## Axios in Production

### Why Axios over Native `fetch`?

| Feature | `fetch` API | Axios |
|:---|:---|:---|
| **JSON parsing** | Manual (`res.json()`) | Automatic |
| **HTTP Errors (404/500)** | Does not reject Promise; check `res.ok` | Automatically rejects Promise |
| **Request Interceptors** | No native support | Yes (global token injection) |
| **Response Interceptors** | No native support | Yes (global 401 handling/refresh) |
| **Request Timeout** | Manual `AbortSignal.timeout()` | Simple `timeout: 5000` option |
| **Upload Progress** | Requires `XMLHttpRequest` | Built-in `onUploadProgress` |

```bash
npm install axios
```

---

### Configuring a Centralized Axios Instance

```javascript
// src/api/apiClient.js
import axios from 'axios';

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'https://api.myapp.com/v1',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
    Accept: 'application/json',
  },
});
```

---

### Interceptors for Auth Tokens & Token Refresh

Interceptors run automatically before requests go out or before responses are handed back to components:

```javascript
// src/api/apiClient.js (Continued)
import { apiClient } from './apiClient';

// 1. Request Interceptor: Attach JWT token to every outgoing request
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// 2. Response Interceptor: Handle global 401 Unauthorized / Token Expiry
apiClient.interceptors.response.use(
  (response) => response.data, // Unpack .data automatically
  async (error) => {
    const originalRequest = error.config;

    // Detect expired token (401) and prevent infinite retry loops
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const refreshToken = localStorage.getItem('refreshToken');
        const res = await axios.post('/api/auth/refresh', { refreshToken });
        const newAccessToken = res.data.token;

        localStorage.setItem('token', newAccessToken);
        originalRequest.headers.Authorization = `Bearer ${newAccessToken}`;

        // Retry original failed request with the new token
        return apiClient(originalRequest);
      } catch (refreshError) {
        // Refresh token failed -> Force user logout
        localStorage.clear();
        window.location.href = '/login';
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);
```

---

## Building a Reusable `useFetch` Custom Hook

```jsx
import { useState, useEffect } from 'react';

export function useFetch(url, options = {}) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(url, { ...options, signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}: ${res.statusText}`);
        return res.json();
      })
      .then(setData)
      .catch(err => {
        if (err.name !== 'AbortError') setError(err.message);
      })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
}
```

---

## TanStack Query (React Query) — The Gold Standard

### Why Server State Libraries are Essential

Traditional global state (like Redux) was designed for synchronous client state. Server data is fundamentally different:
- It is asynchronous, cached, and owned by someone else.
- It can become stale at any moment.
- Multiple components might request the same resource simultaneously.

**TanStack Query** solves this with:
1. **Deduplication:** Multiple components requesting the same data share 1 single network request.
2. **Automatic Background Refetching:** Re-fetches stale data on window focus or network reconnect.
3. **Garbage Collection & In-Memory Caching:** Instant UI responses while refreshing in the background.

```bash
npm install @tanstack/react-query @tanstack/react-query-devtools
```

---

### Setup: `QueryClient` & `QueryClientProvider`

```jsx
// src/main.jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import App from './App';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes before data is considered stale
      gcTime: 1000 * 60 * 15,    // 15 minutes before inactive cache is collected
      retry: 2,                  // Retry failed queries 2 times before showing error
      refetchOnWindowFocus: true,
    },
  },
});

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  </React.StrictMode>
);
```

---

### Fetching with `useQuery`

```jsx
import { useQuery } from '@tanstack/react-query';
import { apiClient } from '../api/apiClient';

function fetchProducts(category) {
  return apiClient.get(`/products`, { params: { category } });
}

export function ProductCatalog({ category }) {
  // Query Key: unique array identifier that acts as cache key
  const { data: products, isLoading, isError, error, refetch } = useQuery({
    queryKey: ['products', category],
    queryFn: () => fetchProducts(category),
    enabled: Boolean(category), // Only fetch if category is provided
  });

  if (isLoading) return <div>Loading catalog...</div>;
  if (isError) return <div>Error loading catalog: {error.message}</div>;

  return (
    <div>
      <button onClick={() => refetch()}>Refresh Products</button>
      <ul>
        {products.map(p => (
          <li key={p.id}>{p.title} — ${p.price}</li>
        ))}
      </ul>
    </div>
  );
}
```

---

### Mutations with `useMutation`

Mutations handle create, update, and delete actions (POST, PUT, PATCH, DELETE):

```jsx
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { apiClient } from '../api/apiClient';

export function AddProductForm() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (newProduct) => apiClient.post('/products', newProduct),
    onSuccess: () => {
      // ✅ Invalidate queries matching ['products']: automatically refetches latest list!
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
    onError: (error) => {
      alert(`Failed to add product: ${error.message}`);
    },
  });

  const handleSubmit = (e) => {
    e.preventDefault();
    mutation.mutate({ title: 'Ergonomic Desk', price: 299 });
  };

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Saving...' : 'Add Product'}
      </button>
    </form>
  );
}
```

---

### Query Keys & Cache Invalidation

Query keys should be modeled hierarchically like paths:

```
['todos']                  // All todos
['todos', 'list', filter]  // Filtered todos
['todos', 'detail', id]    // Specific todo item
```

Invalidating `['todos']` will invalidate all child keys matching that prefix!

---

### Pagination & Infinite Scrolling (`useInfiniteQuery`)

```jsx
import { useInfiniteQuery } from '@tanstack/react-query';

export function InfiniteFeed() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: ({ pageParam = 1 }) =>
      fetch(`/api/posts?page=${pageParam}&limit=10`).then(r => r.json()),
    getNextPageParam: (lastPage, allPages) => {
      // Returns next page number or undefined if no more items
      return lastPage.hasMore ? allPages.length + 1 : undefined;
    },
    initialPageParam: 1,
  });

  return (
    <div>
      {data?.pages.map((group, i) => (
        <div key={i}>
          {group.items.map(post => <p key={post.id}>{post.title}</p>)}
        </div>
      ))}

      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? 'Loading more...' : hasNextPage ? 'Load More' : 'No more posts'}
      </button>
    </div>
  );
}
```

---

### Optimistic Updates Pattern

Optimistically render changes in the UI before the network call completes, and automatically rollback if the server rejects:

```jsx
const mutation = useMutation({
  mutationFn: updateTodoOnServer,
  onMutate: async (updatedTodo) => {
    // 1. Cancel outgoing queries so they don't overwrite optimistic update
    await queryClient.cancelQueries({ queryKey: ['todos'] });

    // 2. Snapshot previous value for rollback
    const previousTodos = queryClient.getQueryData(['todos']);

    // 3. Optimistically set the new value in cache
    queryClient.setQueryData(['todos'], (old = []) =>
      old.map(t => (t.id === updatedTodo.id ? { ...t, ...updatedTodo } : t))
    );

    // 4. Return context containing the snapshot
    return { previousTodos };
  },
  onError: (err, newTodo, context) => {
    // Rollback cache to previous snapshot on network error!
    queryClient.setQueryData(['todos'], context.previousTodos);
  },
  onSettled: () => {
    // Always refetch to ensure client is in sync with server truth
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});
```

---

## SWR (Alternative by Vercel)

### `useSWR` Stale-While-Revalidate Basics

SWR by Vercel stands for **Stale-While-Revalidate**: return cached data first (stale), then fetch the new data in the background (revalidate), and finally update the UI.

```bash
npm install swr
```

```jsx
import useSWR from 'swr';

const fetcher = (url) => fetch(url).then(r => r.json());

export function ProfileWidget() {
  const { data, error, isLoading } = useSWR('/api/user/me', fetcher);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Failed to load profile.</div>;

  return <div>Welcome back, {data.name}!</div>;
}
```

---

### SWR vs. TanStack Query Comparison

| Feature | SWR | TanStack Query |
|:---|:---|:---|
| **Bundle Size** | ~4.5 KB | ~13 KB |
| **Mutation API** | Basic `mutate()` function | Comprehensive `useMutation` with lifecycle hooks |
| **DevTools** | Minimal | Dedicated interactive DevTools panel |
| **Offline Support** | Limited | Robust offline mutations & pause/resume |
| **Ecosystem Standard** | Popular in Next.js apps | Industry-wide React standard |

---

## Enterprise API Layer Architecture

Never scatter raw `fetch()` or `axios.get()` URLs across individual component files. Structure your API calls in dedicated domain services:

```
src/
├── api/
│   ├── apiClient.js       # Axios instance + interceptors
│   ├── authApi.js         # login, logout, refreshToken
│   ├── userApi.js         # getUserProfile, updatePreferences
│   └── productApi.js      # getProducts, getProductById, createProduct
├── hooks/
│   ├── useProducts.js     # React Query hooks encapsulating productApi
│   └── useAuth.js
```

#### Example Service Module (`src/api/productApi.js`):
```javascript
import { apiClient } from './apiClient';

export const productApi = {
  getAll: (params) => apiClient.get('/products', { params }),
  getById: (id) => apiClient.get(`/products/${id}`),
  create: (data) => apiClient.post('/products', data),
  update: (id, data) => apiClient.put(`/products/${id}`, data),
  delete: (id) => apiClient.delete(`/products/${id}`),
};
```

#### Encapsulated Hook (`src/hooks/useProducts.js`):
```javascript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { productApi } from '../api/productApi';

export function useProducts(filters) {
  return useQuery({
    queryKey: ['products', filters],
    queryFn: () => productApi.getAll(filters),
  });
}

export function useCreateProduct() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: productApi.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

---

## 🏋️ Practice Exercises

### Exercise 1: TanStack Query CRUD
Build a simple bookmark manager:
- Fetch bookmarks using `useQuery`.
- Add a new bookmark using `useMutation` with automatic cache invalidation.
- Delete a bookmark with optimistic removal from UI.

### Exercise 2: Infinite Scroll Article Feed
Build a feed that loads 10 articles at a time using `useInfiniteQuery`. Attach a "Load More" button or an intersection observer sentinel element.

---

> **Previous:** [← 07 — Forms](./07-forms.md)  
> **Next:** [09 — Styling →](./09-styling.md)
