# 01 — React Fundamentals

> Everything you need to get started with React — from "What is React?" to building your first interactive UI.

---

## Table of Contents

- [What is React?](#what-is-react)
- [The Virtual DOM](#the-virtual-dom)
- [JSX — JavaScript XML](#jsx--javascript-xml)
- [Components](#components)
- [Props](#props)
- [Events](#events)
- [Conditional Rendering](#conditional-rendering)
- [Lists & Keys](#lists--keys)
- [Fragments](#fragments)
- [Children Props](#children-props)
- [Strict Mode](#strict-mode)

---

## What is React?

React is a **JavaScript library** (not a framework) for building user interfaces. Created by Facebook (Meta) in 2013.

**Core Philosophy:**
- **Declarative** — You describe *what* the UI should look like, React figures out *how* to update the DOM
- **Component-Based** — Build encapsulated, reusable pieces that compose together
- **Learn Once, Write Anywhere** — React Native (mobile), React Three Fiber (3D), etc.

**React vs. Frameworks:**

| Feature | React | Angular | Vue |
|---------|-------|---------|-----|
| Type | Library | Framework | Framework |
| DOM | Virtual DOM | Real DOM (with change detection) | Virtual DOM |
| Language | JSX (JS + HTML) | TypeScript | HTML Templates |
| Routing | External (React Router) | Built-in | External (Vue Router) |
| State Mgmt | External (Redux, Zustand) | Built-in (Services/RxJS) | External (Pinia) |

---

## The Virtual DOM

The Virtual DOM is a lightweight JavaScript representation of the real DOM.

```
  Real DOM                    Virtual DOM
  ┌──────────┐               ┌──────────┐
  │ <div>    │               │ { type:  │
  │  <h1>    │    ◄────►     │   'div', │
  │  <p>     │               │   props, │
  │ </div>   │               │   children }│
  └──────────┘               └──────────┘
```

**How it works:**

1. State changes → React creates a **new** Virtual DOM tree
2. React **diffs** the new tree against the previous one (Reconciliation)
3. React computes the **minimum set of changes** needed
4. React **batches** and applies those changes to the real DOM

**Why is this fast?**
- JavaScript object comparison is much cheaper than DOM manipulation
- Batched updates minimize reflows/repaints
- React's diffing algorithm runs in O(n) time

---

## JSX — JavaScript XML

JSX lets you write HTML-like syntax inside JavaScript. It's syntactic sugar for `React.createElement()`.

### Basic JSX

```jsx
// JSX (what you write)
const element = <h1 className="title">Hello, World!</h1>;

// What it compiles to (Babel/SWC)
const element = React.createElement(
  'h1',
  { className: 'title' },
  'Hello, World!'
);
```

### JSX Rules

```jsx
function App() {
  const name = "Abhi";
  const isLoggedIn = true;

  return (
    // ✅ Rule 1: Must return a SINGLE root element
    <div>
      {/* ✅ Rule 2: Use curly braces {} for JS expressions */}
      <h1>Hello, {name}!</h1>

      {/* ✅ Rule 3: className instead of class */}
      <p className="subtitle">Welcome back</p>

      {/* ✅ Rule 4: htmlFor instead of for */}
      <label htmlFor="email">Email</label>

      {/* ✅ Rule 5: camelCase for attributes */}
      <button onClick={() => alert('clicked')} tabIndex={0}>
        Click Me
      </button>

      {/* ✅ Rule 6: Self-closing tags must close */}
      <img src="photo.jpg" alt="Photo" />
      <input type="text" />
      <br />

      {/* ✅ Rule 7: Inline styles use objects with camelCase */}
      <p style={{ color: 'red', fontSize: '16px', marginTop: '10px' }}>
        Styled text
      </p>

      {/* ✅ Rule 8: JS expressions, NOT statements */}
      <p>{isLoggedIn ? 'Welcome!' : 'Please log in'}</p>
      {/* ❌ Can't do: {if (isLoggedIn) { ... }} */}
    </div>
  );
}
```

### Embedding Expressions

```jsx
function Profile() {
  const user = { name: "Abhi", age: 25 };
  const skills = ["React", "Node.js", "Java"];

  return (
    <div>
      {/* Variables */}
      <h1>{user.name}</h1>

      {/* Calculations */}
      <p>Born in {2026 - user.age}</p>

      {/* Function calls */}
      <p>{user.name.toUpperCase()}</p>

      {/* Template literals */}
      <p>{`${user.name} is ${user.age} years old`}</p>

      {/* Array methods */}
      <p>Skills: {skills.join(', ')}</p>
    </div>
  );
}
```

---

## Components

Components are the building blocks of React applications. Think of them as **custom HTML elements**.

### Function Components (Modern Standard)

```jsx
// Simple component
function Greeting() {
  return <h1>Hello, World!</h1>;
}

// Arrow function (also common)
const Greeting = () => {
  return <h1>Hello, World!</h1>;
};

// Implicit return (single expression)
const Greeting = () => <h1>Hello, World!</h1>;
```

### Component Rules

```jsx
// ✅ Component names MUST start with uppercase
function MyButton() { ... }    // ✅ React component
function myButton() { ... }    // ❌ Treated as HTML element

// ✅ Must return JSX (or null)
function Loading() {
  return null; // Valid — renders nothing
}

// ✅ One component per concern
function UserAvatar() { ... }
function UserName() { ... }
function UserCard() {
  return (
    <div>
      <UserAvatar />
      <UserName />
    </div>
  );
}
```

### Component Composition

```jsx
// Header.jsx
function Header() {
  return (
    <header>
      <Logo />
      <Navigation />
      <SearchBar />
    </header>
  );
}

// App.jsx — composing everything together
function App() {
  return (
    <div className="app">
      <Header />
      <main>
        <Sidebar />
        <Content />
      </main>
      <Footer />
    </div>
  );
}
```

### Class Components (Legacy — Know for Existing Codebases)

```jsx
import { Component } from 'react';

class Greeting extends Component {
  render() {
    return <h1>Hello, {this.props.name}!</h1>;
  }
}

// With state
class Counter extends Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
  }

  increment = () => {
    this.setState(prev => ({ count: prev.count + 1 }));
  };

  render() {
    return (
      <div>
        <p>Count: {this.state.count}</p>
        <button onClick={this.increment}>+</button>
      </div>
    );
  }
}
```

> ⚠️ **Class components are legacy.** All new React code should use function components with hooks. However, you'll encounter class components in older codebases, so know the syntax.

---

## Props

Props (properties) are how you **pass data from parent to child** components. They are **read-only**.

### Basic Props

```jsx
// Parent passes props
function App() {
  return (
    <div>
      <UserCard name="Abhi" age={25} isAdmin={true} />
      <UserCard name="Ravi" age={30} isAdmin={false} />
    </div>
  );
}

// Child receives props
function UserCard(props) {
  return (
    <div className="card">
      <h2>{props.name}</h2>
      <p>Age: {props.age}</p>
      {props.isAdmin && <span className="badge">Admin</span>}
    </div>
  );
}
```

### Destructuring Props (Preferred)

```jsx
// Destructure in parameter
function UserCard({ name, age, isAdmin }) {
  return (
    <div className="card">
      <h2>{name}</h2>
      <p>Age: {age}</p>
      {isAdmin && <span className="badge">Admin</span>}
    </div>
  );
}
```

### Default Props

```jsx
// Method 1: Default parameters (recommended)
function Button({ label = "Click Me", variant = "primary", size = "md" }) {
  return (
    <button className={`btn btn-${variant} btn-${size}`}>
      {label}
    </button>
  );
}

// Usage
<Button />                          // Uses all defaults
<Button label="Submit" />           // Override label only
<Button variant="danger" size="lg" /> // Override variant and size
```

### Passing Different Types as Props

```jsx
function Dashboard() {
  const user = { name: "Abhi", role: "developer" };
  const tags = ["react", "node", "typescript"];

  return (
    <ProfileCard
      // String
      title="Developer Profile"

      // Number
      experience={5}

      // Boolean (these two are equivalent)
      isVerified={true}
      isVerified  // shorthand for true

      // Object
      user={user}

      // Array
      tags={tags}

      // Function
      onEdit={() => console.log('editing...')}

      // JSX / React Element
      icon={<StarIcon />}

      // Spread props
      {...user}  // equivalent to name="Abhi" role="developer"
    />
  );
}
```

### Props Are Read-Only

```jsx
function Broken({ name }) {
  // ❌ NEVER mutate props
  name = "Something else"; // This is wrong!

  // ✅ Use a local variable if you need to transform
  const displayName = name.toUpperCase();

  return <h1>{displayName}</h1>;
}
```

---

## Events

React uses **synthetic events** that wrap native browser events for cross-browser compatibility.

### Basic Event Handling

```jsx
function EventExamples() {
  // Click
  const handleClick = () => {
    console.log('Button clicked!');
  };

  // Click with event object
  const handleClickWithEvent = (e) => {
    console.log('Button text:', e.target.textContent);
    e.preventDefault(); // Prevent default behavior
  };

  // Input change
  const handleChange = (e) => {
    console.log('Input value:', e.target.value);
  };

  // Form submit
  const handleSubmit = (e) => {
    e.preventDefault(); // Prevent page reload
    console.log('Form submitted!');
  };

  return (
    <div>
      <button onClick={handleClick}>Click Me</button>
      <button onClick={handleClickWithEvent}>Click With Event</button>

      <input type="text" onChange={handleChange} />

      <form onSubmit={handleSubmit}>
        <button type="submit">Submit</button>
      </form>
    </div>
  );
}
```

### Passing Arguments to Event Handlers

```jsx
function ItemList() {
  const items = ['Apple', 'Banana', 'Cherry'];

  // Method 1: Arrow function in JSX
  const handleDelete = (item) => {
    console.log(`Deleting ${item}`);
  };

  // Method 2: Curried function (cleaner for complex cases)
  const handleAction = (item) => (e) => {
    console.log(`Action on ${item}`, e.type);
  };

  return (
    <ul>
      {items.map(item => (
        <li key={item}>
          {item}
          {/* Method 1 */}
          <button onClick={() => handleDelete(item)}>Delete</button>
          {/* Method 2 */}
          <button onClick={handleAction(item)}>Action</button>
        </li>
      ))}
    </ul>
  );
}
```

### Common Events Reference

```jsx
function EventsReference() {
  return (
    <div>
      {/* Mouse Events */}
      <button onClick={fn}>Click</button>
      <button onDoubleClick={fn}>Double Click</button>
      <div onMouseEnter={fn}>Mouse Enter</div>
      <div onMouseLeave={fn}>Mouse Leave</div>
      <div onMouseMove={fn}>Mouse Move</div>

      {/* Keyboard Events */}
      <input onKeyDown={fn} />   {/* Key pressed */}
      <input onKeyUp={fn} />     {/* Key released */}

      {/* Form Events */}
      <input onChange={fn} />    {/* Value changed */}
      <input onFocus={fn} />     {/* Focused */}
      <input onBlur={fn} />      {/* Lost focus */}
      <form onSubmit={fn}>...</form>

      {/* Clipboard Events */}
      <input onCopy={fn} onPaste={fn} onCut={fn} />

      {/* Drag Events */}
      <div draggable onDragStart={fn} onDragEnd={fn} onDrop={fn} />

      {/* Scroll */}
      <div onScroll={fn} style={{ overflow: 'auto' }}>...</div>
    </div>
  );
}
```

---

## Conditional Rendering

### All the Patterns

```jsx
function ConditionalExamples({ isLoggedIn, role, notifications, error }) {

  // Pattern 1: if/else (before return)
  if (error) {
    return <ErrorPage message={error} />;
  }

  // Pattern 2: Early return / Guard clause
  if (!isLoggedIn) {
    return <LoginPage />;
  }

  return (
    <div>
      {/* Pattern 3: Ternary (inline, for two branches) */}
      <h1>{role === 'admin' ? 'Admin Dashboard' : 'User Dashboard'}</h1>

      {/* Pattern 4: Logical AND (inline, for one branch) */}
      {notifications.length > 0 && (
        <span className="badge">{notifications.length}</span>
      )}

      {/* Pattern 5: Logical OR (fallback) */}
      <p>{username || 'Anonymous User'}</p>

      {/* Pattern 6: Nullish Coalescing (null/undefined only) */}
      <p>{score ?? 'No score yet'}</p>
      {/* 0 ?? 'No score' → 0 (keeps 0) */}
      {/* 0 || 'No score' → 'No score' (treats 0 as falsy) */}

      {/* Pattern 7: IIFE (complex logic inline) */}
      {(() => {
        switch (role) {
          case 'admin': return <AdminPanel />;
          case 'editor': return <EditorPanel />;
          default: return <ViewerPanel />;
        }
      })()}

      {/* Pattern 8: Object/Map lookup (cleanest for multiple branches) */}
      {
        {
          admin: <AdminPanel />,
          editor: <EditorPanel />,
          viewer: <ViewerPanel />,
        }[role]
      }
    </div>
  );
}
```

### ⚠️ Common Gotcha with `&&`

```jsx
// ❌ BUG: If count is 0, it renders "0" on screen!
{count && <span>{count} items</span>}

// ✅ FIX: Explicitly convert to boolean
{count > 0 && <span>{count} items</span>}
{!!count && <span>{count} items</span>}
{Boolean(count) && <span>{count} items</span>}
```

---

## Lists & Keys

### Rendering Lists

```jsx
function UserList() {
  const users = [
    { id: 1, name: 'Abhi', role: 'Developer' },
    { id: 2, name: 'Ravi', role: 'Designer' },
    { id: 3, name: 'Sara', role: 'Manager' },
  ];

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>
          <strong>{user.name}</strong> — {user.role}
        </li>
      ))}
    </ul>
  );
}
```

### Keys — Why They Matter

Keys help React identify which items have changed, been added, or removed.

```jsx
// ✅ Best: Use stable unique IDs from your data
{users.map(user => <UserCard key={user.id} user={user} />)}

// ⚠️ OK for static lists: Use index (if items never reorder/delete)
{colors.map((color, index) => <span key={index}>{color}</span>)}

// ❌ BAD: Random keys cause full re-mount every render
{users.map(user => <UserCard key={Math.random()} user={user} />)}

// ❌ BAD: Index as key for dynamic lists (causes bugs with state)
{items.map((item, i) => <TodoItem key={i} item={item} />)}
// If you delete item at index 2, item at index 3 becomes index 2,
// and React thinks it's the same component — state gets mixed up!
```

### Rendering Nested Lists

```jsx
function CategoryList() {
  const categories = [
    { id: 1, name: 'Frontend', skills: ['React', 'Vue', 'Angular'] },
    { id: 2, name: 'Backend', skills: ['Node', 'Spring', 'Django'] },
  ];

  return (
    <div>
      {categories.map(cat => (
        <div key={cat.id}>
          <h3>{cat.name}</h3>
          <ul>
            {cat.skills.map(skill => (
              <li key={skill}>{skill}</li>
            ))}
          </ul>
        </div>
      ))}
    </div>
  );
}
```

---

## Fragments

Fragments let you group elements without adding extra DOM nodes.

```jsx
import { Fragment } from 'react';

// Method 1: <Fragment> (when you need keys)
function GlossaryList({ items }) {
  return (
    <dl>
      {items.map(item => (
        <Fragment key={item.id}>
          <dt>{item.term}</dt>
          <dd>{item.definition}</dd>
        </Fragment>
      ))}
    </dl>
  );
}

// Method 2: Short syntax <> </> (when you don't need keys)
function UserInfo({ name, email }) {
  return (
    <>
      <h1>{name}</h1>
      <p>{email}</p>
    </>
  );
}
```

---

## Children Props

The `children` prop allows components to act as **wrappers/containers**.

```jsx
// Reusable Card wrapper
function Card({ title, children }) {
  return (
    <div className="card">
      <div className="card-header">
        <h3>{title}</h3>
      </div>
      <div className="card-body">
        {children}
      </div>
    </div>
  );
}

// Usage — anything between opening/closing tags becomes `children`
function App() {
  return (
    <Card title="User Profile">
      <img src="avatar.jpg" alt="avatar" />
      <p>Abhi — Full Stack Developer</p>
      <button>Edit Profile</button>
    </Card>
  );
}
```

### Multiple Slots with Named Props

```jsx
function Layout({ header, sidebar, children, footer }) {
  return (
    <div className="layout">
      <header>{header}</header>
      <div className="layout-body">
        <aside>{sidebar}</aside>
        <main>{children}</main>
      </div>
      <footer>{footer}</footer>
    </div>
  );
}

// Usage
<Layout
  header={<Navbar />}
  sidebar={<SideMenu />}
  footer={<FooterLinks />}
>
  <h1>Main Content Here</h1>
  <p>This goes into {children}</p>
</Layout>
```

---

## Strict Mode

`StrictMode` is a development-only tool that helps find potential problems in your app.

```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

**What it does (dev only, no effect in production):**

| Check | What it catches |
|-------|----------------|
| Double-invokes render | Impure rendering logic (side effects in render) |
| Double-invokes effects | Missing cleanup in `useEffect` |
| Warns on deprecated APIs | `findDOMNode`, string refs, legacy context |

> 💡 **"Why is my component rendering twice?"** — This is Strict Mode! It's intentional in development. Your component should produce the same result both times. If it doesn't, you have a bug.

---

## 🏋️ Practice Exercises

1. **Profile Card**: Create a `ProfileCard` component that takes `name`, `bio`, `avatarUrl`, and `isOnline` as props. Show a green dot if online.

2. **Todo List**: Build a static todo list. Map over an array of todos. Each todo has `id`, `text`, and `completed`. Strike through completed items.

3. **Navigation**: Create a `NavItem` component and a `Navbar` that renders a list of `NavItem`s. Highlight the active item.

4. **Conditional Banner**: Build a `Banner` component that accepts a `type` prop (`success`, `warning`, `error`, `info`) and renders different colors/icons accordingly.

---

> **Next:** [02 — State & Lifecycle →](./02-state-and-lifecycle.md)
