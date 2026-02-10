# Memoization: When Optimization Becomes Pessimization

**The truth nobody tells you:** Most `useMemo`, `useCallback`, and `React.memo` in your codebase are making things slower, not faster.

## The Fundamental Misconception

**You think:** Memoization makes things faster.

**Reality:** Memoization trades memory for CPU. Sometimes that's a good trade. Often it's not.

Every `useMemo` and `useCallback` adds:
- Memory to store the cached value
- Comparison logic on every render
- Code complexity

**The question isn't "should I memoize?"**
**The question is "is this computation expensive enough to justify the overhead?"**

## The Performance Tax

```javascript
// Without memoization
const filtered = users.filter(u => u.active);

// With memoization
const filtered = useMemo(
  () => users.filter(u => u.active),
  [users]
);
```

**What you pay for `useMemo`:**
1. Function call overhead
2. Dependency comparison (shallow equality check on `[users]`)
3. Memory to store previous users array
4. Memory to store previous result

**When you win:** The filter operation is more expensive than this overhead.
**When you lose:** The filter operation is cheaper than this overhead.

**Spoiler:** Filtering 10 items? You lose. Filtering 10,000? You win.

## The Three Tools

### useMemo - "Don't recalculate this"

**Use when:**
- Expensive calculations (sorting, filtering large datasets)
- Creating objects/arrays that are passed as props to memoized components
- Preventing unnecessary re-renders down the tree

**Don't use when:**
- Simple operations (string concatenation, basic math)
- The component doesn't re-render often
- The computed value changes frequently anyway

```javascript
// ❌ Overkill
const fullName = useMemo(
  () => `${firstName} ${lastName}`,
  [firstName, lastName]
);

// ✅ Just compute it
const fullName = `${firstName} ${lastName}`;

// ❌ List is small
const filtered = useMemo(
  () => users.filter(u => u.active),
  [users]
);  // Overhead > benefit for < 100 items

// ✅ List is large (1000+ items)
const filtered = useMemo(
  () => users.filter(u => u.active),
  [users]
);  // Benefit > overhead
```

### useCallback - "Keep this function reference stable"

**Use when:**
- Passing callbacks to memoized child components
- Dependency in useEffect/useMemo that shouldn't trigger re-run
- Callback is passed to many children

**Don't use when:**
- You're not having re-render issues
- The child isn't memoized
- It's easier to just inline it

```javascript
// ❌ Child isn't memoized, so useCallback does nothing
function Parent() {
  const handleClick = useCallback(() => {
    doSomething();
  }, []);
  
  return <Child onClick={handleClick} />;  // Child re-renders anyway
}

// ✅ Child is memoized, stable callback prevents re-render
const MemoChild = React.memo(Child);

function Parent() {
  const handleClick = useCallback(() => {
    doSomething();
  }, []);
  
  return <MemoChild onClick={handleClick} />;  // Prevents re-render
}

// ❌ useCallback without memoized children = wasted work
function List({ items }) {
  const handleClick = useCallback((id) => {
    console.log(id);
  }, []);
  
  return items.map(item => (
    <Item key={item.id} onClick={handleClick} />  // Not memoized!
  ));
}

// ✅ useCallback + React.memo = worth it
const MemoItem = React.memo(Item);

function List({ items }) {
  const handleClick = useCallback((id) => {
    console.log(id);
  }, []);
  
  return items.map(item => (
    <MemoItem key={item.id} onClick={handleClick} />
  ));
}
```

### React.memo - "Don't re-render unless props changed"

**Use when:**
- Component is expensive to render
- Component re-renders often with same props
- Component is in a list (map)

**Don't use when:**
- Props change frequently
- Component is cheap to render
- You're memoizing everything "just in case"

```javascript
// ❌ Props always change
const User = React.memo(function User({ user, timestamp }) {
  return <div>{user.name} - {timestamp}</div>;
});

// timestamp changes every second → memo is useless, just overhead

// ✅ Props rarely change
const ExpensiveChart = React.memo(function Chart({ data }) {
  // Heavy D3 rendering
  return <svg>...</svg>;
});

// Only re-renders when data changes, not when parent re-renders
```

## The Real-World Scenarios

### Scenario 1: Filtering a List

```javascript
function UserList({ users, searchQuery }) {
  // ❌ Always memoize filtering?
  const filtered = useMemo(
    () => users.filter(u => u.name.includes(searchQuery)),
    [users, searchQuery]
  );
  
  return filtered.map(user => <UserCard key={user.id} user={user} />);
}
```

**Analysis:**
- **If `users.length < 100`:** The filter is fast. `useMemo` overhead is not worth it.
- **If `searchQuery` changes on every keystroke:** The memo cache is invalidated constantly. No benefit.
- **If `users.length > 1000` AND `searchQuery` is debounced:** NOW memoization helps.

**Better:**
```javascript
function UserList({ users, searchQuery }) {
  // Debounce first, then decide on memoization
  const debouncedQuery = useDebounce(searchQuery, 300);
  
  // Only memoize if users.length > 1000
  const filtered = users.length > 1000
    ? useMemo(
        () => users.filter(u => u.name.includes(debouncedQuery)),
        [users, debouncedQuery]
      )
    : users.filter(u => u.name.includes(debouncedQuery));
  
  return filtered.map(user => <UserCard key={user.id} user={user} />);
}
```

### Scenario 2: Context Provider Value

```javascript
// ❌ WITHOUT useMemo - re-renders all consumers on every render
function MyProvider({ children }) {
  const [user, setUser] = useState(null);
  const [settings, setSettings] = useState({});
  
  return (
    <MyContext.Provider value={{ user, setUser, settings, setSettings }}>
      {children}
    </MyContext.Provider>
  );
}

// ✅ WITH useMemo - only re-renders when dependencies change
function MyProvider({ children }) {
  const [user, setUser] = useState(null);
  const [settings, setSettings] = useState({});
  
  const value = useMemo(
    () => ({ user, setUser, settings, setSettings }),
    [user, settings]
  );
  
  return (
    <MyContext.Provider value={value}>
      {children}
    </MyContext.Provider>
  );
}
```

**This is one of the FEW cases where you ALWAYS want useMemo.**

### Scenario 3: Expensive Calculation

```javascript
function PrimeNumbers({ limit }) {
  // ❌ Without memo - recalculates on EVERY render (even parent re-renders)
  const primes = calculatePrimes(limit);  // Takes 500ms for limit=1000000
  
  return <div>{primes.length} primes found</div>;
}

// ✅ With memo - only recalculates when limit changes
function PrimeNumbers({ limit }) {
  const primes = useMemo(
    () => calculatePrimes(limit),
    [limit]
  );
  
  return <div>{primes.length} primes found</div>;
}
```

**Guideline:** If a calculation takes >50ms, memoize it.

### Scenario 4: List with Callbacks

```javascript
// ❌ Every item re-renders on every parent render
function TodoList({ todos, onToggle }) {
  return todos.map(todo => (
    <TodoItem 
      key={todo.id}
      todo={todo}
      onToggle={onToggle}  // Might be new function each render
    />
  ));
}

// ❌ Adding React.memo alone doesn't help if onToggle changes
const MemoTodoItem = React.memo(TodoItem);

function TodoList({ todos, onToggle }) {
  return todos.map(todo => (
    <MemoTodoItem 
      key={todo.id}
      todo={todo}
      onToggle={onToggle}  // Still a new function!
    />
  ));
}

// ✅ useCallback + React.memo = only changed todos re-render
function TodoApp() {
  const [todos, setTodos] = useState([]);
  
  const handleToggle = useCallback((id) => {
    setTodos(prev => prev.map(t => 
      t.id === id ? { ...t, done: !t.done } : t
    ));
  }, []);  // Stable function
  
  return <TodoList todos={todos} onToggle={handleToggle} />;
}

const MemoTodoItem = React.memo(TodoItem);

function TodoList({ todos, onToggle }) {
  return todos.map(todo => (
    <MemoTodoItem 
      key={todo.id}
      todo={todo}
      onToggle={onToggle}
    />
  ));
}
```

**Key insight:** `React.memo` + `useCallback` work together. One without the other is often useless.

## The Benchmarking Truth

**Don't guess. Measure.**

```javascript
function MyComponent() {
  // Measure without memoization
  console.time('calculation');
  const result = expensiveCalculation(data);
  console.timeEnd('calculation');  // "calculation: 0.2ms"
  
  // If < 5ms, DON'T memoize. The overhead isn't worth it.
}
```

**Chrome DevTools Profiler** is your friend:
1. Open Profiler tab
2. Record interaction
3. Look at component render times
4. Yellow/red = optimize these
5. Blue/green = leave alone

## The Memoization Decision Tree

```
Is this causing a performance problem?
├─ No → Don't memoize
└─ Yes
    │
    What's the issue?
    ├─ Expensive calculation
    │   └─ Does it take >50ms?
    │       ├─ Yes → useMemo
    │       └─ No → Don't memoize
    │
    ├─ Child re-rendering unnecessarily
    │   └─ Is the child expensive to render?
    │       ├─ Yes → React.memo on child + useCallback for props
    │       └─ No → Fix component structure instead
    │
    └─ Context causing re-renders
        └─ Always useMemo the context value
```

## The Anti-Patterns

### 1. Memoizing Everything

```javascript
// ❌ Cargo-cult memoization
function Component({ user, products, settings }) {
  const userName = useMemo(() => user.name, [user.name]);
  const productCount = useMemo(() => products.length, [products.length]);
  const isDarkMode = useMemo(() => settings.theme === 'dark', [settings.theme]);
  
  // You just made this component SLOWER
}
```

**Reality:** These operations are instant. The memoization overhead is greater than the computation.

### 2. useMemo with Missing Dependencies

```javascript
// ❌ Stale data bug
function Component({ userId }) {
  const [filter, setFilter] = useState('');
  
  const data = useMemo(
    () => fetchUserData(userId, filter),
    [userId]  // Missing filter!
  );
  
  // Changing filter doesn't refetch because it's not in dependencies
}
```

**Fix:** Use React Query/SWR. Don't memoize async data fetching.

### 3. useCallback with Everything in Dependencies

```javascript
// ❌ useCallback that never caches
function Component({ data, config, user, settings }) {
  const handleClick = useCallback(() => {
    processData(data, config, user, settings);
  }, [data, config, user, settings]);  // Changes on every render!
  
  // The callback is never stable, so it's useless overhead
}
```

### 4. Memoizing and Then Not Using the Memo

```javascript
// ❌ Child isn't memoized
function Parent() {
  const value = useMemo(() => expensiveCalc(), []);
  return <Child value={value} />;  // Child re-renders anyway
}

// ✅ Child is memoized
const MemoChild = React.memo(Child);

function Parent() {
  const value = useMemo(() => expensiveCalc(), []);
  return <MemoChild value={value} />;  // Now it helps
}
```

## The Guidelines

**useMemo:**
- ✅ Expensive calculations (>50ms)
- ✅ Context provider values
- ✅ Creating objects/arrays for memoized children
- ❌ Simple operations
- ❌ Values that change every render

**useCallback:**
- ✅ Callbacks passed to memoized children
- ✅ Dependencies in useEffect that shouldn't trigger
- ✅ Event handlers in lists (with memoized items)
- ❌ Inline event handlers on non-memoized components
- ❌ "Just in case"

**React.memo:**
- ✅ Expensive components
- ✅ List items (map)
- ✅ Components that re-render often with same props
- ❌ Components where props change frequently
- ❌ Cheap components
- ❌ Components with children prop (children always changes)

## The Profiling Workflow

```
1. Write code normally (no memoization)
   ↓
2. Test the feature
   ↓
3. Is there a noticeable lag?
   ├─ No → Ship it
   └─ Yes ↓
4. Open React DevTools Profiler
   ↓
5. Record the slow interaction
   ↓
6. Identify the bottleneck
   ├─ Expensive calculation → useMemo
   ├─ Unnecessary re-renders → React.memo + useCallback
   └─ Too many items → Virtualization (react-window)
   ↓
7. Apply targeted optimization
   ↓
8. Re-measure to confirm improvement
   ↓
9. Ship it
```

**Don't skip step 1.** Write normal code first. Optimize later.

## The Bottom Line

**Premature optimization is the root of all evil.**

Every hook you add is:
- One more thing to maintain
- One more place for bugs
- One more mental overhead

**Optimize when you have evidence of a problem, not because you think there might be one someday.**

**The React team's advice:**
> "You don't need to optimize every component. React is fast by default."

Most apps never need more than 5-10 well-placed memoizations.

If you have 100+ useMemo/useCallback calls, you're doing it wrong.

---

### Further Reading
- [Before You memo()](https://overreacted.io/before-you-memo/) by Dan Abramov
- [useMemo/useCallback Optimization](https://kentcdodds.com/blog/usememo-and-usecallback) by Kent C. Dodds
- [React DevTools Profiler](https://react.dev/reference/react/Profiler)