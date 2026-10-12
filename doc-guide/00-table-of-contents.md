# 📚 React.js — Complete Study Guide

> **From Zero to Production-Ready** — A comprehensive, hands-on guide covering every React concept you'll need for building real-world applications.

---

## 🗂️ Guide Structure

| # | Module | Key Topics |
|---|--------|------------|
| 01 | [Fundamentals](./01-fundamentals.md) | JSX, Components, Props, Events, Conditional Rendering, Lists & Keys |
| 02 | [State & Lifecycle](./02-state-and-lifecycle.md) | `useState`, Component Lifecycle, `useEffect`, Cleanup |
| 03 | [Hooks Deep Dive](./03-hooks-deep-dive.md) | All built-in hooks, Custom Hooks, Rules of Hooks |
| 04 | [Advanced Patterns](./04-advanced-patterns.md) | HOCs, Render Props, Compound Components, Portals, Error Boundaries |
| 05 | [State Management](./05-state-management.md) | Context API, Redux Toolkit, Zustand, Jotai, Recoil |
| 06 | [Routing](./06-routing.md) | React Router v6+, Nested Routes, Guards, Lazy Routes |
| 07 | [Forms](./07-forms.md) | Controlled/Uncontrolled, React Hook Form, Formik, Yup/Zod Validation |
| 08 | [Data Fetching](./08-data-fetching.md) | fetch, Axios, TanStack Query, SWR, Suspense for Data |
| 09 | [Styling](./09-styling.md) | CSS Modules, Styled Components, Tailwind CSS, CSS-in-JS |
| 10 | [Testing](./10-testing.md) | Vitest, React Testing Library, MSW, E2E with Playwright |
| 11 | [Performance](./11-performance-optimization.md) | `React.memo`, `useMemo`, `useCallback`, Code Splitting, Virtualization |
| 12 | [TypeScript + React](./12-typescript-with-react.md) | Typing Props, Hooks, Events, Generics, Utility Types |
| 13 | [Real-World Patterns](./13-real-world-patterns.md) | Project Structure, Auth, Error Handling, API Layers, Environment Config |
| 14 | [Ecosystem & Tools](./14-ecosystem-and-tools.md) | Vite, Next.js, UI Libraries, Animation, Essential Packages |

---

## 🎯 How to Use This Guide

### If you're a beginner:
1. Start with **01-Fundamentals** → **02-State & Lifecycle** → **03-Hooks Deep Dive**
2. Then move to **06-Routing** → **07-Forms** → **08-Data Fetching**
3. Circle back to **05-State Management** and **04-Advanced Patterns**

### If you're intermediate:
1. Skim **01-03** for any gaps
2. Focus on **04-Advanced Patterns**, **05-State Management**, **11-Performance**
3. Deep dive into **12-TypeScript** and **13-Real-World Patterns**

### If you're prepping for a project:
1. Read **13-Real-World Patterns** first for architecture decisions
2. Pick your stack from **14-Ecosystem & Tools**
3. Reference specific modules as you build

---

## 🛠️ Prerequisites

- **JavaScript ES6+** — destructuring, spread, arrow functions, promises, async/await, modules
- **HTML/CSS** — basic understanding
- **Node.js & npm** — installed on your machine
- **Code Editor** — VS Code recommended

---

## ⚡ Quick Project Setup

```bash
# Create a new React project with Vite (recommended)
npm create vite@latest my-react-app -- --template react

# With TypeScript
npm create vite@latest my-react-app -- --template react-ts

# Navigate and install
cd my-react-app
npm install
npm run dev
```

---

> 💡 **Every section includes code examples that you can copy-paste and run.** Concepts are explained with real-world context, not just theory.
