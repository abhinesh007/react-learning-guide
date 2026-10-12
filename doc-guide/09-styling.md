# 09 — Styling in React

> Explore the full spectrum of styling paradigms in the React ecosystem: CSS Modules, Tailwind CSS, CSS-in-JS (Styled Components, Emotion), CSS Variables, and responsive theming architectures.

---

## Table of Contents

- [The Styling Landscape](#the-styling-landscape)
- [Inline Styles](#inline-styles)
  - [Syntax & CamelCase Rules](#syntax--camelcase-rules)
  - [Pros, Cons & Performance Limitations](#pros-cons--performance-limitations)
- [Plain CSS & Global Stylesheets](#plain-css--global-stylesheets)
  - [Global Collisions & Specificity Wars](#global-collisions--specificity-wars)
- [CSS Modules (Zero-Runtime Scoping)](#css-modules-zero-runtime-scoping)
  - [How CSS Modules Work](#how-css-modules-work)
  - [Usage in Vite / Create React App](#usage-in-vite--create-react-app)
  - [Class Composition (`composes`)](#class-composition-composes)
- [CSS-in-JS: Styled Components](#css-in-js-styled-components)
  - [Tagged Template Literals](#tagged-template-literals)
  - [Dynamic Styling Based on Props](#dynamic-styling-based-on-props)
  - [Theming via `<ThemeProvider>`](#theming-via-themeprovider)
  - [Runtime Performance Trade-offs](#runtime-performance-trade-offs)
- [CSS-in-JS: Emotion](#css-in-js-emotion)
  - [The `css` Prop & `styled` Syntax](#the-css-prop--styled-syntax)
- [Tailwind CSS (Utility-First Modern Standard)](#tailwind-css-utility-first-modern-standard)
  - [Vite Integration](#vite-integration)
  - [Conditional Classes with `clsx` and `tailwind-merge`](#conditional-classes-with-clsx-and-tailwind-merge)
  - [The `cn()` Utility Helper Function](#the-cn-utility-helper-function)
  - [Building Reusable Component Primitives](#building-reusable-component-primitives)
- [Modern CSS Variables & Dark Mode Architecture](#modern-css-variables--dark-mode-architecture)
- [Comprehensive Styling Comparison Matrix](#comprehensive-styling-comparison-matrix)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## The Styling Landscape

Styling in React has evolved through three distinct eras:
1. **Plain CSS / Preprocessors (SASS/LESS):** Great familiarity, but vulnerable to global namespace collisions and selector bloat.
2. **CSS-in-JS (Styled Components, Emotion):** Colocation and dynamic prop-driven styles, but introduces client-side runtime overhead and SSR complexity.
3. **Zero-Runtime & Utility-First (CSS Modules, Tailwind CSS, StyleX):** Maximum performance, predictable styling, and no client-side CSS generation overhead.

---

## Inline Styles

### Syntax & CamelCase Rules

Inline styles in React are written as JavaScript objects passed to the `style` attribute. CSS properties with hyphens are camelCased:

```jsx
export function AlertBox({ type, message }) {
  const isError = type === 'error';

  const alertStyle = {
    backgroundColor: isError ? '#fee2e2' : '#dcfce7',
    color: isError ? '#991b1b' : '#166534',
    padding: '16px',
    borderRadius: '8px',
    fontSize: '14px',
    fontWeight: 500,
    marginTop: '12px',
  };

  return <div style={alertStyle}>{message}</div>;
}
```

### Pros, Cons & Performance Limitations

| Advantages | Limitations |
|:---|:---|
| Completely dynamic with React state/props | No pseudo-classes (`:hover`, `:focus`, `:active`) |
| No external CSS files needed | No media queries (`@media (max-width: 768px)`) |
| Scoped directly to the individual element | No keyframe animations |
| Highest specificity (hard to override) | Generates new object reference on every render unless memoized |

> 💡 **Rule:** Use inline styles only for purely dynamic values that change at 60fps (e.g., cursor coordinates, slider drag positions, dynamic transforms).

---

## Plain CSS & Global Stylesheets

Importing a standard `.css` file into a React component makes all classes globally available across the entire web application:

```jsx
// src/components/Button.jsx
import './Button.css';

export function Button({ label }) {
  return <button className="btn-primary">{label}</button>;
}
```

### Global Collisions & Specificity Wars

If `Card.jsx` also defines a `.btn-primary` class inside its own imported CSS file, one class will overwrite the other depending on bundle evaluation order. Plain CSS requires disciplined naming conventions like **BEM** (`block__element--modifier`) to avoid accidental overrides:

```css
/* BEM Naming Convention */
.c-product-card {}
.c-product-card__title {}
.c-product-card__button--disabled {}
```

---

## CSS Modules (Zero-Runtime Scoping)

### How CSS Modules Work

CSS Modules scope CSS class names locally by hashing the class name during build time (e.g., transforming `.title` into `.Button_title__a8d3f`). This eliminates naming collisions completely with **zero runtime overhead**.

### Usage in Vite / Create React App

Simply name the file with `.module.css`:

```css
/* src/components/Badge.module.css */
.badge {
  display: inline-flex;
  align-items: center;
  padding: 4px 8px;
  border-radius: 9999px;
  font-size: 12px;
  font-weight: 600;
}

.success {
  background-color: #d1fae5;
  color: #065f46;
}

.warning {
  background-color: #fef3c7;
  color: #92400e;
}
```

```jsx
// src/components/Badge.jsx
import styles from './Badge.module.css';

export function Badge({ status, text }) {
  const statusClass = status === 'success' ? styles.success : styles.warning;

  return (
    <span className={`${styles.badge} ${statusClass}`}>
      {text}
    </span>
  );
}
```

---

### Class Composition (`composes`)

CSS Modules allow you to compose rules from other classes without duplicating declarations:

```css
/* src/components/Typography.module.css */
.baseText {
  font-family: system-ui, sans-serif;
  line-height: 1.5;
}

.heading {
  composes: baseText;
  font-size: 24px;
  font-weight: 700;
  color: #111827;
}
```

---

## CSS-in-JS: Styled Components

Styled Components allows writing actual CSS syntax inside tagged template literals directly in your JavaScript files.

```bash
npm install styled-components
```

### Tagged Template Literals & Props

```jsx
import styled, { css } from 'styled-components';

export const StyledButton = styled.button`
  background: transparent;
  border-radius: 6px;
  border: 2px solid #3b82f6;
  color: #3b82f6;
  padding: 8px 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;

  &:hover {
    background: #3b82f6;
    color: white;
  }

  /* Dynamic conditional styling based on component props */
  ${props => props.$variant === 'danger' && css`
    border-color: #ef4444;
    color: #ef4444;
    &:hover {
      background: #ef4444;
      color: white;
    }
  `}

  ${props => props.$size === 'lg' && css`
    padding: 12px 24px;
    font-size: 18px;
  `}
`;

// Usage:
// <StyledButton $variant="danger" $size="lg">Delete Account</StyledButton>
```

> 💡 **Tip:** Use transient props prefixed with `$` (like `$variant`) so they aren't passed down to the real underlying HTML DOM element.

---

### Theming via `<ThemeProvider>`

```jsx
import { ThemeProvider, createGlobalStyle } from 'styled-components';

const theme = {
  colors: {
    primary: '#6366f1',
    background: '#ffffff',
    text: '#1f2937',
  },
  spacing: (multiplier) => `${multiplier * 8}px`,
};

const GlobalStyle = createGlobalStyle`
  body {
    margin: 0;
    background-color: ${props => props.theme.colors.background};
    color: ${props => props.theme.colors.text};
    font-family: Inter, sans-serif;
  }
`;

export function App() {
  return (
    <ThemeProvider theme={theme}>
      <GlobalStyle />
      <MainContent />
    </ThemeProvider>
  );
}
```

---

### Runtime Performance Trade-offs

Styled Components injects `<style>` tags dynamically at runtime on every render cycle when prop-based styles change. In apps with high-frequency updates (e.g. data tables, animation loops), this causes style recalculation overhead and layout thrashing.

---

## CSS-in-JS: Emotion

Emotion is an alternative CSS-in-JS library featuring the flexible `css` prop and high runtime performance:

```bash
npm install @emotion/react @emotion/styled
```

```jsx
/** @jsxImportSource @emotion/react */
import { css } from '@emotion/react';

const cardStyle = css`
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 20px;
  &:hover {
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
  }
`;

export function Card({ children }) {
  return <div css={cardStyle}>{children}</div>;
}
```

---

## Tailwind CSS (Utility-First Modern Standard)

Tailwind CSS compiles utility classes on-demand into an ultra-lean CSS file, eliminating runtime JS overhead.

### Vite Integration

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

```javascript
// tailwind.config.js
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        brand: {
          50: '#eef2ff',
          500: '#6366f1',
          600: '#4f46e5',
        },
      },
    },
  },
  plugins: [],
};
```

---

### Conditional Classes with `clsx` and `tailwind-merge`

When creating reusable React components that accept an external `className` prop, class collisions can occur (e.g. `p-4` passed to a component with default `p-2`).

- `clsx`: handles conditional classes (`isActive && 'bg-blue-500'`).
- `tailwind-merge`: resolves Tailwind class conflicts in favor of the incoming override.

```bash
npm install clsx tailwind-merge
```

---

### The `cn()` Utility Helper Function

Standard in modern design systems (such as `shadcn/ui`):

```javascript
// src/lib/utils.js
import { clsx } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs) {
  return twMerge(clsx(inputs));
}
```

---

### Building Reusable Component Primitives

```jsx
// src/components/Button.jsx
import { cn } from '../lib/utils';

export function Button({
  variant = 'primary',
  size = 'md',
  className,
  children,
  ...props
}) {
  const baseStyles = 'inline-flex items-center justify-center font-medium rounded-md transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2 disabled:opacity-50';

  const variants = {
    primary: 'bg-indigo-600 text-white hover:bg-indigo-700 focus:ring-indigo-500',
    secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200 focus:ring-gray-400',
    danger: 'bg-red-600 text-white hover:bg-red-700 focus:ring-red-500',
    outline: 'border border-gray-300 bg-transparent hover:bg-gray-50 focus:ring-gray-400',
  };

  const sizes = {
    sm: 'text-xs px-2.5 py-1.5',
    md: 'text-sm px-4 py-2',
    lg: 'text-base px-6 py-3',
  };

  return (
    <button
      className={cn(baseStyles, variants[variant], sizes[size], className)}
      {...props}
    >
      {children}
    </button>
  );
}

// Consumer overrides default padding cleanly:
// <Button variant="primary" className="p-8">Huge Button</Button>
```

---

## Modern CSS Variables & Dark Mode Architecture

CSS Custom Properties (Variables) allow instant runtime theming with zero re-renders of the React component tree:

```css
/* index.css */
:root {
  --bg-primary: #ffffff;
  --text-primary: #111827;
  --card-bg: #f9fafb;
  --border-color: #e5e7eb;
}

[data-theme='dark'] {
  --bg-primary: #0f172a;
  --text-primary: #f8fafc;
  --card-bg: #1e293b;
  --border-color: #334155;
}

body {
  background-color: var(--bg-primary);
  color: var(--text-primary);
  transition: background-color 0.2s ease, color 0.2s ease;
}
```

```jsx
// src/components/ThemeToggle.jsx
import { useState, useEffect } from 'react';

export function ThemeToggle() {
  const [theme, setTheme] = useState(() => localStorage.getItem('theme') || 'light');

  useEffect(() => {
    document.documentElement.setAttribute('data-theme', theme);
    localStorage.setItem('theme', theme);
  }, [theme]);

  return (
    <button onClick={() => setTheme(t => (t === 'light' ? 'dark' : 'light'))}>
      Switch to {theme === 'light' ? 'Dark 🌙' : 'Light ☀️'}
    </button>
  );
}
```

---

## Comprehensive Styling Comparison Matrix

| Criteria | CSS Modules | Styled Components | Tailwind CSS |
|:---|:---|:---|:---|
| **Runtime Overhead** | 0 KB | ~12 KB + recalculations | 0 KB |
| **CSS Scoping** | Build-time hash | Runtime generated class | Utility classes |
| **Design Consistency** | Manual | Manual / ThemeProvider | Built-in design tokens |
| **Developer Speed** | Moderate | Moderate | Extremely Fast |
| **Dead Code Elimination** | Moderate | Good | Automated (PurgeCSS) |
| **Industry Trend** | Steady | Decreasing | Rapidly Growing |

---

## 🏋️ Practice Exercises

### Exercise 1: Dark Mode with CSS Variables & Tailwind
Configure Tailwind's `darkMode: 'class'` and create a persistent theme toggle component that detects the user's OS preference via `window.matchMedia('(prefers-color-scheme: dark)')`.

### Exercise 2: Composable Component Library Card
Build a `<Card>` component with `<Card.Header>`, `<Card.Body>`, and `<Card.Footer>` styled with Tailwind CSS and the `cn()` utility, allowing arbitrary consumer overrides.

---

> **Previous:** [← 08 — Data Fetching](./08-data-fetching.md)  
> **Next:** [10 — Testing →](./10-testing.md)
