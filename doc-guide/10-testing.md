# 10 — Testing React Applications

> The complete testing playbook for React: Vitest, React Testing Library, user-event, Mock Service Worker (MSW), testing custom hooks, and Playwright end-to-end testing.

---

## Table of Contents

- [The Modern Testing Philosophy](#the-modern-testing-philosophy)
  - [The Testing Trophy vs. Testing Pyramid](#the-testing-trophy-vs-testing-pyramid)
  - [Test User Behavior, Not Implementation Details](#test-user-behavior-not-implementation-details)
- [Test Environment Setup (Vitest + RTL)](#test-environment-setup-vitest--rtl)
  - [Installation & Vite Config](#installation--vite-config)
  - [Configuring the JSDOM Setup File](#configuring-the-jsdom-setup-file)
- [React Testing Library (RTL) Core Mechanics](#react-testing-library-rtl-core-mechanics)
  - [The Query Priority Hierarchy](#the-query-priority-hierarchy)
  - [`userEvent` vs. `fireEvent`](#userevent-vs-fireevent)
  - [Handling Asynchronous Elements: `waitFor` & `findBy`](#handling-asynchronous-elements-waitfor--findby)
- [Core Testing Patterns & Examples](#core-testing-patterns--examples)
  - [Testing Static & Conditional Rendering](#testing-static--conditional-rendering)
  - [Testing User Interactions & Form Submissions](#testing-user-interactions--form-submissions)
  - [Testing Custom Hooks with `renderHook`](#testing-custom-hooks-with-renderhook)
- [Mocking in React](#mocking-in-react)
  - [Mocking API Calls with Mock Service Worker (MSW)](#mocking-api-calls-with-mock-service-worker-msw)
  - [Mocking Module Dependencies with `vi.mock`](#mocking-module-dependencies-with-vimock)
  - [Mocking React Router Navigation](#mocking-react-router-navigation)
  - [Testing with Context Providers](#testing-with-context-providers)
- [Snapshot Testing: When Useful vs. Fragile](#snapshot-testing-when-useful-vs-fragile)
- [End-to-End Testing with Playwright](#end-to-end-testing-with-playwright)
  - [Setup & Architecture](#setup--architecture)
  - [Writing a Real Authentication E2E Test](#writing-a-real-authentication-e2e-test)
- [Code Coverage Metrics](#code-coverage-metrics)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## The Modern Testing Philosophy

### The Testing Trophy vs. Testing Pyramid

Historically, development teams prioritized unit testing every individual function in isolation. In modern React development, Kent C. Dodds' **Testing Trophy** model is favored:

```
          ╱  E2E  ╲         Few: Critical smoke paths (Playwright)
        ╱───────────╲
      ╱  INTEGRATION  ╲     Most: Components interacting with state, router & APIs
    ╱───────────────────╲
   │        UNIT         │   Some: Complex standalone helper calculations, reducers
   └─────────────────────┘
   │       STATIC        │   All: TypeScript compiler, ESLint, Prettier
   └─────────────────────┘
```

Integration tests provide the highest confidence-to-effort ratio because they test components the way real humans interact with them.

---

### Test User Behavior, Not Implementation Details

#### ❌ Fragile Test (Tests internal implementation details):
```javascript
// BAD: Asserts on component state or internal class names
expect(wrapper.state('count')).toBe(1);
expect(wrapper.find('div.btn-blue')).toHaveLength(1);
```
*Why this is bad:* Refactoring class names or replacing `useState` with `useReducer` breaks the test even if the UI still works perfectly for the user.

#### ✅ Resilient Test (Tests user observable behavior):
```javascript
// GOOD: Clicks the visible button labeled "Increment", asserts visible text updates
const button = screen.getByRole('button', { name: /increment/i });
await user.click(button);
expect(screen.getByText(/count: 1/i)).toBeInTheDocument();
```

---

## Test Environment Setup (Vitest + RTL)

### Installation & Vite Config

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.js',
    css: false, // Skip CSS processing for faster tests
  },
});
```

---

### Configuring the JSDOM Setup File

```javascript
// src/test/setup.js
import '@testing-library/jest-dom';
import { afterEach } from 'vitest';
import { cleanup } from '@testing-library/react';

// Clean up DOM after each individual test run
afterEach(() => {
  cleanup();
});
```

Add test script to `package.json`:
```json
"scripts": {
  "test": "vitest",
  "test:run": "vitest run",
  "test:coverage": "vitest run --coverage"
}
```

---

## React Testing Library (RTL) Core Mechanics

### The Query Priority Hierarchy

RTL provides queries designed to reflect how users find elements on screen. Always follow this priority order:

| Priority | Query Function | Explanation / Real World Analogy |
|:---|:---|:---|
| **1 (Top)** | `getByRole` | Accessible tree lookup (buttons, headings, inputs). Best for a11y! |
| **2** | `getByLabelText` | Finding form inputs associated with `<label>` |
| **3** | `getByPlaceholderText` | Form inputs when no label is present |
| **4** | `getByText` | Finding static text content (paragraphs, divs) |
| **5** | `getByDisplayValue` | Current value of a form element |
| **6 (Escape Hatch)** | `getByTestId` | `data-testid` attribute when elements have no accessible role |

```
Query Prefix Matrix:
- getBy...     --> Throws error immediately if element is NOT found.
- queryBy...   --> Returns null if not found. (Use ONLY when asserting non-existence!)
- findBy...    --> Returns a Promise. Waits up to 1000ms for element to appear asynchronously.
```

---

### `userEvent` vs. `fireEvent`

- `fireEvent`: Dispatches artificial DOM events (`fireEvent.click(btn)`). Does not trigger focus, keydown, keyup, or hover events.
- `userEvent`: Simulates full realistic browser user interaction chains (typing character by character, blurring inputs, firing key events). **Always prefer `userEvent`!**

```javascript
import userEvent from '@testing-library/user-event';

test('typing in input', async () => {
  const user = userEvent.setup(); // Always initialize setup()
  render(<SearchInput />);
  
  const input = screen.getByRole('textbox', { name: /search/i });
  await user.type(input, 'React testing');
  expect(input).toHaveValue('React testing');
});
```

---

### Handling Asynchronous Elements: `waitFor` & `findBy`

```javascript
import { render, screen, waitFor } from '@testing-library/react';

test('displays loaded notification message', async () => {
  render(<AsyncNotification />);

  // Option A: Using findBy (shorthand for waitFor + getBy)
  const alert = await screen.findByRole('alert');
  expect(alert).toHaveTextContent(/update completed/i);

  // Option B: Using waitFor for complex assertions
  await waitFor(() => {
    expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
  });
});
```

---

## Core Testing Patterns & Examples

### Testing Static & Conditional Rendering

```jsx
// src/components/Greeting.jsx
export function Greeting({ user }) {
  if (!user) {
    return <p>Please sign in to continue.</p>;
  }
  return <h1>Welcome back, {user.name}!</h1>;
}
```

```javascript
// src/components/Greeting.test.jsx
import { render, screen } from '@testing-library/react';
import { Greeting } from './Greeting';

describe('Greeting component', () => {
  test('renders sign-in prompt when no user is provided', () => {
    render(<Greeting user={null} />);
    expect(screen.getByText(/please sign in to continue/i)).toBeInTheDocument();
    expect(screen.queryByRole('heading')).not.toBeInTheDocument();
  });

  test('renders user welcome heading when user is logged in', () => {
    render(<Greeting user={{ name: 'Abhi' }} />);
    expect(screen.getByRole('heading', { level: 1 })).toHaveTextContent('Welcome back, Abhi!');
  });
});
```

---

### Testing User Interactions & Form Submissions

```javascript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { vi } from 'vitest';
import { LoginForm } from './LoginForm';

test('submits user credentials when form is valid', async () => {
  const user = userEvent.setup();
  const handleLoginMock = vi.fn();

  render(<LoginForm onLogin={handleLoginMock} />);

  const emailInput = screen.getByLabelText(/email address/i);
  const passwordInput = screen.getByLabelText(/password/i);
  const submitButton = screen.getByRole('button', { name: /sign in/i });

  await user.type(emailInput, 'user@example.com');
  await user.type(passwordInput, 'SecretPassword123');
  await user.click(submitButton);

  expect(handleLoginMock).toHaveBeenCalledTimes(1);
  expect(handleLoginMock).toHaveBeenCalledWith({
    email: 'user@example.com',
    password: 'SecretPassword123',
  });
});
```

---

### Testing Custom Hooks with `renderHook`

Custom hooks cannot be executed directly outside React component execution. Use `renderHook` and `act`:

```javascript
// src/hooks/useCounter.test.js
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

test('increments and resets counter state', () => {
  const { result } = renderHook(() => useCounter(10));

  expect(result.current.count).toBe(10);

  // Wrap state-modifying actions inside act()
  act(() => {
    result.current.increment();
  });
  expect(result.current.count).toBe(11);

  act(() => {
    result.current.reset();
  });
  expect(result.current.count).toBe(10);
});
```

---

## Mocking in React

### Mocking API Calls with Mock Service Worker (MSW)

MSW intercepts network calls at the network level rather than monkey-patching `fetch`, providing high fidelity tests:

```bash
npm install -D msw
```

```javascript
// src/test/server.js
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('https://api.example.com/products', () => {
    return HttpResponse.json([
      { id: 1, title: 'Ergonomic Chair', price: 199 },
      { id: 2, title: 'USB Hub', price: 29 },
    ]);
  }),
];

export const server = setupServer(...handlers);
```

```javascript
// src/test/setup.js
import { server } from './server';

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

```javascript
// src/components/ProductCatalog.test.jsx
import { render, screen } from '@testing-library/react';
import { ProductCatalog } from './ProductCatalog';

test('fetches and renders remote products', async () => {
  render(<ProductCatalog />);

  expect(screen.getByText(/loading products/i)).toBeInTheDocument();

  // Elements will appear once MSW responds
  expect(await screen.findByText('Ergonomic Chair')).toBeInTheDocument();
  expect(await screen.findByText('USB Hub')).toBeInTheDocument();
});
```

---

### Mocking Module Dependencies with `vi.mock`

```javascript
import { vi } from 'vitest';

// Mock an entire analytics library
vi.mock('../services/analytics', () => ({
  trackEvent: vi.fn(),
}));

import { trackEvent } from '../services/analytics';
```

---

### Mocking React Router Navigation

```javascript
import { MemoryRouter } from 'react-router-dom';
import { render, screen } from '@testing-library/react';

// Use MemoryRouter to provide routing context in tests
render(
  <MemoryRouter initialEntries={['/products/42']}>
    <ProductDetailRoute />
  </MemoryRouter>
);
```

---

### Testing with Context Providers

Create a custom render utility that wraps components with all necessary providers:

```javascript
// src/test/test-utils.jsx
import { render } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { AuthProvider } from '../context/AuthContext';

export function renderWithProviders(ui, options = {}) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  function Wrapper({ children }) {
    return (
      <QueryClientProvider client={queryClient}>
        <AuthProvider>
          {children}
        </AuthProvider>
      </QueryClientProvider>
    );
  }

  return render(ui, { wrapper: Wrapper, ...options });
}
```

---

## Snapshot Testing: When Useful vs. Fragile

```javascript
test('matches snapshot', () => {
  const { asFragment } = render(<PriceTag amount={49.99} currency="USD" />);
  expect(asFragment()).toMatchSnapshot();
});
```

- **When Useful:** Small, stable design tokens, icons, or complex math layout formatters that rarely change.
- **When Fragile:** Large page components. Developers often mindlessly run `vitest -u` to update broken snapshots without verifying whether the markup change was intentional.

---

## End-to-End Testing with Playwright

Playwright runs tests in real Chromium, Firefox, and WebKit browsers.

### Setup & Architecture

```bash
npm init playwright@latest
```

```javascript
// e2e/auth.spec.js
import { test, expect } from '@playwright/test';

test.describe('Authentication Flow', () => {
  test('allows an existing user to log in and access dashboard', async ({ page }) => {
    // 1. Visit application
    await page.goto('http://localhost:5173/login');

    // 2. Fill login form
    await page.getByLabel('Email').fill('admin@company.com');
    await page.getByLabel('Password').fill('SuperSecret123');
    await page.getByRole('button', { name: 'Sign In' }).click();

    // 3. Verify redirected to dashboard
    await expect(page).toHaveURL('http://localhost:5173/dashboard');
    await expect(page.getByRole('heading', { name: 'Welcome to your Dashboard' })).toBeVisible();
  });
});
```

---

## Code Coverage Metrics

Run `vitest run --coverage` using V8 or Istanbul. Focus on **branch coverage** across business logic rather than aiming blindly for 100% line coverage on trivial boilerplate.

---

## 🏋️ Practice Exercises

### Exercise 1: Test a Registration Form
Write a complete suite for a signup component:
- Validates that error messages appear when clicking submit with empty inputs.
- Checks that the submit callback is invoked with correct values when valid data is entered.

### Exercise 2: Mock API & Test Error States
Using MSW, write tests for:
1. Successful loading of user profile data.
2. Server error (500) scenario ensuring the user sees an accessible alert error banner with a "Retry" button.

---

> **Previous:** [← 09 — Styling](./09-styling.md)  
> **Next:** [11 — Performance Optimization →](./11-performance-optimization.md)
