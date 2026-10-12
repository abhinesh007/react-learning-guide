# 13 — Real-World Project Architecture & Patterns

> The blueprint for production React applications: feature-based folder architecture, JWT authentication cycles, API service layers, global error boundaries, modals, toasts, i18n, and CI/CD deployment.

---

## Table of Contents

- [Production Project Architecture](#production-project-architecture)
  - [Feature-Based vs. Type-Based Structures](#feature-based-vs-type-based-structures)
  - [Enterprise Folder Blueprint](#enterprise-folder-blueprint)
  - [Barrel Exports (`index.ts`) & Path Aliasing](#barrel-exports-indexts--path-aliasing)
- [Environment Configuration](#environment-configuration)
  - [Vite `.env` Conventions (`VITE_` Prefix)](#vite-env-conventions-vite_-prefix)
  - [Build-Time vs. Runtime Configuration](#build-time-vs-runtime-configuration)
  - [Validating Environment Variables with Zod](#validating-environment-variables-with-zod)
- [Enterprise API Service Layer](#enterprise-api-service-layer)
  - [Centralized Axios Client & Interceptors](#centralized-axios-client--interceptors)
  - [Domain-Specific Service Modules](#domain-specific-service-modules)
- [Production Authentication Flow](#production-authentication-flow)
  - [JWT Architecture (Access & Refresh Tokens)](#jwt-architecture-access--refresh-tokens)
  - [Session Persistence & Silent Refresh](#session-persistence--silent-refresh)
  - [Automatic Idle Logout](#automatic-idle-logout)
- [Resilient Error Handling Strategy](#resilient-error-handling-strategy)
  - [Layered Error Boundaries](#layered-error-boundaries)
  - [Toast Alerts for API Failures](#toast-alerts-for-api-failures)
- [Loading States & Skeleton Screens](#loading-states--skeleton-screens)
  - [Shimmer Skeletons vs. Spinner Hell](#shimmer-skeletons-vs-spinner-hell)
- [Toast & Notification Systems](#toast--notification-systems)
  - [Using `sonner` / `react-hot-toast`](#using-sonner--react-hot-toast)
- [Modal & Dialog Management](#modal--dialog-management)
  - [Portal Confirmation Dialog Hook](#portal-confirmation-dialog-hook)
- [Dark Mode & Theming](#dark-mode--theming)
- [Internationalization (i18n)](#internationalization-i18n)
  - [Setting up `react-i18next`](#setting-up-react-i18next)
- [SEO & Meta Tags in SPAs](#seo--meta-tags-in-spas)
  - [Using `react-helmet-async`](#using-react-helmet-async)
- [Production Deployment](#production-deployment)
  - [Vercel & Netlify SPA Redirects (`_redirects`)](#vercel--netlify-spa-redirects-_redirects)
  - [Production Multi-Stage Dockerfile](#production-multi-stage-dockerfile)
- [Code Quality Tooling](#code-quality-tooling)
  - [ESLint, Prettier, Husky & lint-staged](#eslint-prettier-husky--lint-staged)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## Production Project Architecture

### Feature-Based vs. Type-Based Structures

In smaller toy projects, files are grouped by *type* (`components/`, `hooks/`, `services/`). In large real-world codebases with dozens of engineers, **feature-based structure** is preferred: everything related to a business domain (components, hooks, API calls, types) is colocated inside that feature's directory.

```
Type-Based (Legacy/Small Apps):
src/
├── components/   # 80 components mixed together
├── hooks/        # 40 hooks mixed together
└── services/     # 20 services

Feature-Based (Scalable Enterprise Standard):
src/
├── features/
│   ├── auth/          # components, hooks, api, types for auth
│   ├── billing/       # components, hooks, api, types for billing
│   └── dashboard/     # components, hooks, api, types for dashboard
└── shared/            # Reusable UI primitives (Button, Modal, Input)
```

---

### Enterprise Folder Blueprint

```
src/
├── assets/                  # Static media, SVGs, brand icons
├── components/              # Shared UI design system primitives
│   ├── ui/                  # Button, Dialog, Input, Card, Badge
│   └── layout/              # Header, Sidebar, Footer, PageWrapper
├── config/                  # App constants, route paths, env config
├── context/                 # Application-wide global contexts
├── features/                # Domain-driven feature modules
│   ├── auth/
│   │   ├── api/             # authApi.ts
│   │   ├── components/      # LoginForm.tsx, SignupCard.tsx
│   │   ├── hooks/           # useAuthSession.ts
│   │   ├── types/           # auth.types.ts
│   │   └── index.ts         # Public barrel export
│   └── products/
│       ├── api/productApi.ts
│       ├── components/ProductCard.tsx
│       └── hooks/useProducts.ts
├── hooks/                   # App-wide utility hooks (useDebounce, useMediaQuery)
├── lib/                     # Third-party wrapper configs (axios, queryClient, utils)
├── routes/                  # AppRoutes.tsx, ProtectedRoute.tsx
├── types/                   # Global shared TypeScript models
└── main.tsx                 # Root entry
```

---

### Barrel Exports (`index.ts`) & Path Aliasing

#### Barrel Exports:
Create an `index.ts` in each feature folder to expose only its public API, keeping internals private:

```typescript
// src/features/auth/index.ts
export * from './components/LoginForm';
export * from './hooks/useAuthSession';
export * from './types/auth.types';
// Do NOT export internal unshared helper components
```

#### Path Aliasing in `tsconfig.json` & `vite.config.ts`:
Eliminate brittle `../../../../` relative import paths:

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

```typescript
// vite.config.ts
import path from 'path';
import { defineConfig } from 'vite';

export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

Now import cleanly anywhere:
```typescript
import { Button } from '@/components/ui/Button';
import { useAuth } from '@/features/auth';
```

---

## Environment Configuration

### Vite `.env` Conventions (`VITE_` Prefix)

By default, Vite only exposes environment variables prefixed with `VITE_` to the client browser to prevent accidentally leaking private secrets (like database passwords or AWS keys) into public JS bundles:

```bash
# .env.development
VITE_API_BASE_URL=http://localhost:4000/api
VITE_ENABLE_ANALYTICS=false

# .env.production
VITE_API_BASE_URL=https://api.myproductiondomain.com/api
VITE_ENABLE_ANALYTICS=true
```

Access in application code:
```typescript
const apiBase = import.meta.env.VITE_API_BASE_URL;
```

---

### Validating Environment Variables with Zod

Ensure your app fails loudly at startup with an explanatory error if a required environment variable is missing:

```typescript
// src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  VITE_API_BASE_URL: z.string().url(),
  VITE_APP_NAME: z.string().default('My Dashboard'),
  VITE_ENABLE_MOCKS: z.enum(['true', 'false']).default('false'),
});

export const env = envSchema.parse(import.meta.env);
```

---

## Enterprise API Service Layer

```typescript
// src/lib/apiClient.ts
import axios from 'axios';
import { env } from '@/config/env';

export const apiClient = axios.create({
  baseURL: env.VITE_API_BASE_URL,
  timeout: 15000,
  headers: { 'Content-Type': 'application/json' },
});

apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

---

## Production Authentication Flow

### JWT Architecture (Access & Refresh Tokens)

```
┌────────────────┐                     ┌────────────────┐
│ React Client   │                     │ Auth Server    │
└───────┬────────┘                     └───────┬────────┘
        │ 1. POST /login (credentials)         │
        ├─────────────────────────────────────►│
        │ 2. Return accessToken + refreshToken  │
        │◄─────────────────────────────────────┤
        │ (Store in Memory / HttpOnly Cookie)  │
        │                                      │
        │ 3. API Call with Bearer Token        │
        ├─────────────────────────────────────►│
        │ 4. HTTP 401 Unauthorized (Expired)   │
        │◄─────────────────────────────────────┤
        │                                      │
        │ 5. POST /refresh (refreshToken)      │
        ├─────────────────────────────────────►│
        │ 6. New accessToken returned          │
        │◄─────────────────────────────────────┤
        │                                      │
        │ 7. Retry original failed API call    │
        ├─────────────────────────────────────►│
        │ 8. HTTP 200 OK                       │
        │◄─────────────────────────────────────┘
```

---

### Automatic Idle Logout

Log users out if no keyboard or mouse activity is detected for 15 minutes:

```typescript
import { useEffect, useRef } from 'react';

export function useIdleTimeout(onIdle: () => void, timeoutMs = 15 * 60 * 1000) {
  const timerRef = useRef<number | undefined>(undefined);

  const resetTimer = () => {
    window.clearTimeout(timerRef.current);
    timerRef.current = window.setTimeout(onIdle, timeoutMs);
  };

  useEffect(() => {
    const events = ['mousemove', 'keydown', 'mousedown', 'touchstart', 'scroll'];
    events.forEach(e => window.addEventListener(e, resetTimer));
    resetTimer();

    return () => {
      window.clearTimeout(timerRef.current);
      events.forEach(e => window.removeEventListener(e, resetTimer));
    };
  }, [timeoutMs]);
}
```

---

## Resilient Error Handling Strategy

### Layered Error Boundaries

Do not rely solely on a single root error boundary. Isolate boundaries around specific widgets:

```jsx
<AppLayout>
  <ErrorBoundary fallback={<HeaderFallback />}>
    <Header />
  </ErrorBoundary>

  <div className="main-content">
    <ErrorBoundary fallback={<SidebarFallback />}>
      <Sidebar />
    </ErrorBoundary>

    <ErrorBoundary fallback={<PageCrashReport />}>
      <Outlet />
    </ErrorBoundary>
  </div>
</AppLayout>
```

If the `Sidebar` crashes due to bad data, the user can still read and interact with the main page content!

---

## Loading States & Skeleton Screens

### Shimmer Skeletons vs. Spinner Hell

Full-screen spinners cause visual disorientation. Skeleton placeholders reduce perceived load time by pre-rendering the layout shapes:

```jsx
// src/components/ui/Skeleton.tsx
export function Skeleton({ className = '' }) {
  return (
    <div
      className={`animate-pulse bg-gray-200 dark:bg-gray-700 rounded ${className}`}
    />
  );
}

// Card with Skeleton State:
export function ProductCardSkeleton() {
  return (
    <div className="border p-4 rounded-lg space-y-3">
      <Skeleton className="h-40 w-full" />
      <Skeleton className="h-6 w-3/4" />
      <Skeleton className="h-4 w-1/2" />
      <div className="flex justify-between pt-2">
        <Skeleton className="h-8 w-20" />
        <Skeleton className="h-8 w-24" />
      </div>
    </div>
  );
}
```

---

## Toast & Notification Systems

### Using `sonner`

`sonner` is the most popular, accessible toast notification library in the modern React ecosystem:

```bash
npm install sonner
```

```jsx
// src/App.tsx
import { Toaster, toast } from 'sonner';

export default function App() {
  return (
    <div>
      <Toaster richColors position="top-right" />
      <Dashboard />
    </div>
  );
}

// In any feature action:
const handleSave = async () => {
  toast.promise(saveDocument(), {
    loading: 'Saving document...',
    success: 'Document saved successfully!',
    error: (err) => `Failed to save: ${err.message}`,
  });
};
```

---

## Modal & Dialog Management

### Portal Confirmation Dialog Hook

```tsx
import { useState, useCallback, ReactNode } from 'react';
import { createPortal } from 'react-dom';

interface ConfirmationOptions {
  title: string;
  message: string;
  confirmLabel?: string;
}

export function useConfirmationModal() {
  const [modalState, setModalState] = useState<{
    isOpen: boolean;
    options: ConfirmationOptions;
    resolve: (value: boolean) => void;
  } | null>(null);

  const confirm = useCallback((options: ConfirmationOptions): Promise<boolean> => {
    return new Promise((resolve) => {
      setModalState({ isOpen: true, options, resolve });
    });
  }, []);

  const handleChoice = (accepted: boolean) => {
    modalState?.resolve(accepted);
    setModalState(null);
  };

  const ModalDialog = modalState?.isOpen ? createPortal(
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
      <div className="bg-white p-6 rounded-lg max-w-sm w-full">
        <h3 className="text-lg font-bold">{modalState.options.title}</h3>
        <p className="mt-2 text-gray-600">{modalState.options.message}</p>
        <div className="mt-6 flex justify-end gap-3">
          <button onClick={() => handleChoice(false)}>Cancel</button>
          <button
            className="bg-red-600 text-white px-4 py-2 rounded"
            onClick={() => handleChoice(true)}
          >
            {modalState.options.confirmLabel || 'Confirm'}
          </button>
        </div>
      </div>
    </div>,
    document.body
  ) : null;

  return { confirm, ModalDialog };
}

// Usage in any component:
// const { confirm, ModalDialog } = useConfirmationModal();
// const proceed = await confirm({ title: 'Delete Item?', message: 'This is irreversible.' });
// if (proceed) deleteItem();
```

---

## Internationalization (i18n)

### Setting up `react-i18next`

```bash
npm install i18next react-i18next i18next-browser-languagedetector
```

```javascript
// src/lib/i18n.js
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';

i18n
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    fallbackLng: 'en',
    resources: {
      en: {
        translation: {
          welcome: 'Welcome, {{name}}',
          logout: 'Sign Out',
        },
      },
      es: {
        translation: {
          welcome: 'Bienvenido, {{name}}',
          logout: 'Cerrar sesión',
        },
      },
    },
  });

export default i18n;
```

```jsx
import { useTranslation } from 'react-i18next';

export function Header({ userName }) {
  const { t, i18n } = useTranslation();

  return (
    <div>
      <p>{t('welcome', { name: userName })}</p>
      <button onClick={() => i18n.changeLanguage('es')}>ES</button>
      <button onClick={() => i18n.changeLanguage('en')}>EN</button>
    </div>
  );
}
```

---

## SEO & Meta Tags in SPAs

### Using `react-helmet-async`

Client-side SPAs render title and meta descriptions on the fly:

```bash
npm install react-helmet-async
```

```jsx
import { Helmet, HelmetProvider } from 'react-helmet-async';

export function ProductPage({ product }) {
  return (
    <div>
      <Helmet>
        <title>{product.name} | Acme Store</title>
        <meta name="description" content={product.description} />
        <meta property="og:title" content={product.name} />
        <meta property="og:image" content={product.imageUrl} />
      </Helmet>
      <h1>{product.name}</h1>
    </div>
  );
}
```

---

## Production Deployment

### Vercel & Netlify SPA Redirects (`_redirects`)

In an SPA, navigating directly to `https://myapp.com/dashboard/settings` will return a 404 from standard web servers because no real `settings.html` file exists on disk. You must redirect all non-file requests to `index.html`:

#### Netlify (`public/_redirects`):
```
/*    /index.html   200
```

#### Vercel (`vercel.json`):
```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

---

### Production Multi-Stage Dockerfile

```dockerfile
# Stage 1: Build static assets
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Serve with lightweight NGINX Alpine
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## Code Quality Tooling

Set up automated Git hooks to format and lint code before commits are accepted:

```bash
npm install -D husky lint-staged prettier eslint
npx husky init
```

```json
// package.json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,css}": [
      "prettier --write"
    ]
  }
}
```

---

## 🏋️ Practice Exercises

### Exercise 1: Architect a Feature Folder
Structure a complete `features/orders/` directory containing:
- `api/orderApi.ts` with typed endpoints
- `hooks/useOrders.ts` consuming TanStack Query
- `components/OrderList.tsx` and `OrderBadge.tsx`
- `index.ts` exporting only the public API.

### Exercise 2: Confirmation Dialog Hook
Implement the `useConfirmationModal` hook and integrate it into a delete button flow with realistic animations.

---

> **Previous:** [← 12 — TypeScript with React](./12-typescript-with-react.md)  
> **Next:** [14 — Ecosystem & Essential Libraries →](./14-ecosystem-and-tools.md)
