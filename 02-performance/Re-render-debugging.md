# Re-render Debugging: Find the Leak, Not the Symptom

**The problem:** Your app feels sluggish. You add `React.memo` everywhere. It's still slow. You don't know why.

**The solution:** Stop guessing. Start measuring.

## The Mental Model

Every re-render happens for exactly 3 reasons:

1. **State changed** (useState, useReducer)
2. **Props changed** (parent re-rendered and passed new values)
3. **Context changed** (provider value changed)

That's it. No magic. If your component is re-rendering, one of these three happened.

## The React DevTools Profiler (Your Best Friend)

**Step 1: Install React DevTools**
- Chrome/Edge/Firefox extension
- Already have it? Update it. Old versions miss features.

**Step 2: Open Profiler Tab**

**Step 3: Click Record → Interact with your app → Stop**

**What you're looking for:**
- **Yellow/red components** = Expensive renders
- **Flame graph height** = Render time depth
- **Ranked chart** = Which components took longest

**Pro tip:** Enable "Record why each component rendered" in settings.

## The Quick Diagnosis

### Scenario 1: Everything Re-renders on Every Keystroke

```javascript
// ❌ The culprit
function App() {
  const [search, setSearch] = useState('');
  
  return (
    <div>
      <SearchBar value={search} onChange={setSearch} />
      <UserList />      {/* Re-renders on every keystroke! */}
      <Analytics />     {/* Re-renders on every keystroke! */}
      <Footer />        {/* Re-renders on every keystroke! */}
    </div>
  );
}
```

**Why?** State change in `App` → `App` re-renders → All children re-render.

**Fix 1: Lift the search state down**
```javascript
// ✅ Isolate the state
function App() {
  return (
    <div>
      <SearchSection />  {/* Only this re-renders */}
      <UserList />
      <Analytics />
      <Footer />
    </div>
  );
}

function SearchSection() {
  const [search, setSearch] = useState('');
  return <SearchBar value={search} onChange={setSearch} />;
}
```

**Fix 2: Use composition**
```javascript
// ✅ Pass children
function App() {
  return (
    <SearchWrapper>
      <UserList />
      <Analytics />
      <Footer />
    </SearchWrapper>
  );
}

function SearchWrapper({ children }) {
  const [search, setSearch] = useState('');
  
  return (
    <div>
      <SearchBar value={search} onChange={setSearch} />
      {children}  {/* These don't re-render! */}
    </div>
  );
}
```

**Why it works:** `children` is a prop. It doesn't change when `SearchWrapper` re-renders.

### Scenario 2: Context Provider Causing Mass Re-renders

```javascript
// ❌ Classic mistake
function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('dark');
  
  const value = { user, setUser, theme, setTheme };  // New object every render!
  
  return (
    <AppContext.Provider value={value}>
      <Dashboard />  {/* Re-renders when user OR theme changes */}
    </AppContext.Provider>
  );
}
```

**Fix: useMemo the value**
```javascript
// ✅ Stable reference
function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('dark');
  
  const value = useMemo(
    () => ({ user, setUser, theme, setTheme }),
    [user, theme]
  );
  
  return (
    <AppContext.Provider value={value}>
      <Dashboard />
    </AppContext.Provider>
  );
}
```

**Better: Split contexts**
```javascript
// ✅ Best solution
function App() {
  return (
    <UserProvider>
      <ThemeProvider>
        <Dashboard />  {/* Only re-renders when the context it uses changes */}
      </ThemeProvider>
    </UserProvider>
  );
}

// Components only re-render when their specific context changes
function Header() {
  const theme = useTheme();  // Only re-renders on theme change
  return <header className={theme}>...</header>;
}

function UserMenu() {
  const user = useUser();  // Only re-renders on user change
  return <div>{user.name}</div>;
}
```

### Scenario 3: List Re-rendering All Items

```javascript
// ❌ Every item re-renders when one changes
function TodoList() {
  const [todos, setTodos] = useState([...]);
  
  const toggle = (id) => {
    setTodos(todos.map(t => 
      t.id === id ? { ...t, done: !t.done } : t
    ));
  };
  
  return todos.map(todo => (
    <TodoItem 
      key={todo.id} 
      todo={todo} 
      onToggle={toggle}  // New function every render!
    />
  ));
}

function TodoItem({ todo, onToggle }) {
  console.log('TodoItem rendered:', todo.id);
  return <div onClick={() => onToggle(todo.id)}>{todo.text}</div>;
}
```

**Problem 1:** New `toggle` function every render → all items re-render
**Problem 2:** Entire `todos` array is new → all items re-render even if individual todo didn't change

**Fix:**
```javascript
// ✅ Stable callbacks + memo
function TodoList() {
  const [todos, setTodos] = useState([...]);
  
  const toggle = useCallback((id) => {
    setTodos(prev => prev.map(t => 
      t.id === id ? { ...t, done: !t.done } : t
    ));
  }, []);  // Stable function
  
  return todos.map(todo => (
    <MemoizedTodoItem 
      key={todo.id} 
      todo={todo} 
      onToggle={toggle} 
    />
  ));
}

const MemoizedTodoItem = React.memo(function TodoItem({ todo, onToggle }) {
  console.log('TodoItem rendered:', todo.id);
  return <div onClick={() => onToggle(todo.id)}>{todo.text}</div>;
});
```

**Now:** Only the changed todo re-renders. Rest stay memoized.

### Scenario 4: Anonymous Functions in Props

```javascript
// ❌ New function every render
function UserList({ users }) {
  return users.map(user => (
    <UserCard 
      key={user.id}
      user={user}
      onClick={() => console.log(user.id)}  // New function!
    />
  ));
}
```

**Fix 1: Extract to component**
```javascript
// ✅ Component handles its own callback
function UserList({ users }) {
  return users.map(user => <UserCard key={user.id} user={user} />);
}

function UserCard({ user }) {
  const handleClick = () => console.log(user.id);  // Stable per user
  return <div onClick={handleClick}>{user.name}</div>;
}
```

**Fix 2: useCallback with dependency**
```javascript
// ✅ When you need to pass the handler up
function UserList({ users, onUserClick }) {
  return users.map(user => (
    <UserCardMemo 
      key={user.id}
      user={user}
      onClick={onUserClick}
    />
  ));
}

const UserCardMemo = React.memo(function UserCard({ user, onClick }) {
  const handleClick = useCallback(() => {
    onClick(user.id);
  }, [user.id, onClick]);
  
  return <div onClick={handleClick}>{user.name}</div>;
});
```

## The Debugging Tools

### 1. Why Did You Render (Library)

```bash
npm install @welldone-software/why-did-you-render
```

```javascript
// wdyr.js
import React from 'react';
import whyDidYouRender from '@welldone-software/why-did-you-render';

if (process.env.NODE_ENV === 'development') {
  whyDidYouRender(React, {
    trackAllPureComponents: true,
    trackHooks: true,
    logOnDifferentValues: true,
  });
}
```

**What it does:** Console logs exactly why each component re-rendered.

**Output:**
```
UserCard re-rendered because:
  - props.onClick changed from function(){} to function(){}
  - props.user.name changed from "John" to "John Doe"
```

### 2. React DevTools "Highlight Updates"

**Settings → Enable "Highlight updates when components render"**

Interacting with your app shows colored borders around re-rendering components:
- **Blue flash** = Infrequent renders
- **Yellow/orange** = Moderate renders
- **Red** = Too many renders

**Use this to spot:** Which part of your UI is re-rendering when you type, scroll, or click.

### 3. Custom useWhyDidYouUpdate Hook

```javascript
function useWhyDidYouUpdate(name, props) {
  const previousProps = useRef();

  useEffect(() => {
    if (previousProps.current) {
      const allKeys = Object.keys({ ...previousProps.current, ...props });
      const changedProps = {};
      
      allKeys.forEach(key => {
        if (previousProps.current[key] !== props[key]) {
          changedProps[key] = {
            from: previousProps.current[key],
            to: props[key]
          };
        }
      });

      if (Object.keys(changedProps).length > 0) {
        console.log('[why-did-you-update]', name, changedProps);
      }
    }

    previousProps.current = props;
  });
}

// Usage
function UserCard({ user, onClick }) {
  useWhyDidYouUpdate('UserCard', { user, onClick });
  return <div>{user.name}</div>;
}
```

## Common Culprits (The Hall of Shame)

### 1. Inline Object/Array Literals
```javascript
// ❌ New object every render
<UserCard user={user} style={{ color: 'red' }} />

// ✅ Define outside or useMemo
const cardStyle = { color: 'red' };
<UserCard user={user} style={cardStyle} />
```

### 2. Inline Arrow Functions
```javascript
// ❌ New function every render
<button onClick={() => doSomething()}>Click</button>

// ✅ useCallback or named function
const handleClick = useCallback(() => doSomething(), []);
<button onClick={handleClick}>Click</button>
```

### 3. Derived State That Should Be Computed
```javascript
// ❌ Triggers re-render when it shouldn't
const [filtered, setFiltered] = useState([]);

useEffect(() => {
  setFiltered(users.filter(u => u.active));
}, [users]);

// ✅ Just compute it
const filtered = useMemo(
  () => users.filter(u => u.active),
  [users]
);
```

### 4. Context with Unstable Values
```javascript
// ❌ New object on every provider re-render
<MyContext.Provider value={{ x: 1, y: 2 }}>

// ✅ Memoize it
const value = useMemo(() => ({ x: 1, y: 2 }), []);
<MyContext.Provider value={value}>
```

### 5. Spreading Props
```javascript
// ❌ Can cause unnecessary re-renders
<Child {...props} />

// ✅ Pass only what's needed
<Child name={props.name} age={props.age} />
```

## The "Should I Optimize?" Checklist

Before you start memoizing everything:

- [ ] Is there a **visible** performance problem? (Measured, not assumed)
- [ ] Did React DevTools Profiler show this component is expensive?
- [ ] Does this component render 100+ times per interaction?
- [ ] Does this component have expensive calculations?

If you answered "no" to all of these: **Don't optimize yet.**

**Premature optimization IS the root of all evil** in React. Memoization adds complexity.

## The Performance Investigation Flow

```
App feels slow
    ↓
Open React DevTools Profiler
    ↓
Record interaction that feels slow
    ↓
Look at flame graph / ranked chart
    ↓
Identify components with:
- Long render times (yellow/red)
- High render counts
    ↓
For each slow component, ask:
    ↓
1. Why is it re-rendering?
   → Check props, state, context
    ↓
2. Is the re-render necessary?
   → Yes: Optimize the render itself (virtualization, lazy loading)
   → No: Prevent the re-render (memo, composition, state location)
    ↓
3. Fix and re-measure
    ↓
Repeat until acceptable
```

## The Quick Wins

**Fastest performance improvements:**

1. **Move state down** - Isolate state to the smallest possible subtree
2. **Split contexts** - One context per concern
3. **Use composition** - Pass `children` instead of re-rendering
4. **Virtualize long lists** - react-window for 100+ items
5. **Lazy load heavy components** - `React.lazy()` for routes/modals

**These will solve 90% of performance issues before you touch a single `useMemo`.**

## The Bottom Line

**Bad:** Sprinkling `React.memo` and `useCallback` everywhere hoping it helps.

**Good:** Measuring with DevTools, identifying the cause, fixing the architecture, then optimizing if still needed.

Most performance problems are **state location problems**, not memoization problems.

Fix your component tree first. Memoize second.

---

### Tools
- [React DevTools](https://react.dev/learn/react-developer-tools)
- [Why Did You Render](https://github.com/welldone-software/why-did-you-render)
- [React DevTools Profiler Guide](https://react.dev/reference/react/Profiler)