# Stale State: The Silent Killer of "It Worked Yesterday"

**The bug:** Your component reads old data even though you just updated it.

**The confusion:** "I literally just set the state... why is it still the old value??"

**The culprit:** JavaScript closures. Every time.

## The Classic Example

```javascript
// ❌ This looks correct but is broken
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      console.log(count); // Always logs 0
      setCount(count + 1); // Always sets to 1
    }, 1000);
    
    return () => clearInterval(interval);
  }, []); // Empty deps = closure captures initial count (0)
  
  return <div>{count}</div>;
}
```

**What you expect:** Count increments every second: 0, 1, 2, 3, 4...

**What happens:** Count goes from 0 → 1, then stays at 1 forever.

**Why:** The interval callback closes over `count = 0`. It never sees the updated value.

## The Mental Model

When you write:
```javascript
useEffect(() => {
  // This function captures variables from THIS render
  const handler = () => {
    console.log(count); // Captures count from when effect ran
  };
}, []);
```

You're creating a **time capsule**. That function remembers the values from when it was created.

**Empty dependency array = time capsule from first render only.**

## Pattern 1: Stale State in Intervals/Timers

### The Bug:
```javascript
function ChatInput() {
  const [message, setMessage] = useState('');
  
  useEffect(() => {
    // Auto-save draft every 5 seconds
    const interval = setInterval(() => {
      saveDraft(message); // Always saves initial message ('')
    }, 5000);
    
    return () => clearInterval(interval);
  }, []); // Missing dependency!
  
  return <input value={message} onChange={e => setMessage(e.target.value)} />;
}
```

**User types:** "Hello world"  
**Auto-save saves:** ""  
**User:** "WTF"

### Fix Option 1: Add to Dependencies
```javascript
// ✅ Effect re-runs when message changes
useEffect(() => {
  const interval = setInterval(() => {
    saveDraft(message); // Always sees latest
  }, 5000);
  
  return () => clearInterval(interval);
}, [message]); // Re-create interval when message changes
```

**Problem:** This restarts the interval on EVERY keystroke. Not ideal.

### Fix Option 2: Use Ref for Latest Value
```javascript
// ✅ Keep interval stable, read latest value
function ChatInput() {
  const [message, setMessage] = useState('');
  const messageRef = useRef(message);
  
  // Keep ref in sync
  useEffect(() => {
    messageRef.current = message;
  }, [message]);
  
  useEffect(() => {
    const interval = setInterval(() => {
      saveDraft(messageRef.current); // Always reads latest
    }, 5000);
    
    return () => clearInterval(interval);
  }, []); // Stable interval
  
  return <input value={message} onChange={e => setMessage(e.target.value)} />;
}
```

### Fix Option 3: Use Functional Updates (If Setting State)
```javascript
// ✅ Best for state updates
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      setCount(prev => prev + 1); // Reads latest from React
    }, 1000);
    
    return () => clearInterval(interval);
  }, []); // No dependency needed
  
  return <div>{count}</div>;
}
```

**Rule:** If you're **setting** state based on previous state, use the function form. If you're **reading** state for other purposes, use a ref.

## Pattern 2: Stale State in Event Handlers

### The Bug:
```javascript
function SearchBar() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    // Add global keyboard shortcut
    const handleKeyPress = (e) => {
      if (e.key === '/') {
        search(query); // Always searches initial query ('')
      }
    };
    
    document.addEventListener('keypress', handleKeyPress);
    return () => document.removeEventListener('keypress', handleKeyPress);
  }, []); // Missing query dependency
  
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

**User types:** "react hooks"  
**User presses:** `/`  
**Searches for:** ""  
**User:** Confused rage

### The Fix:
```javascript
// ✅ Include query in dependencies
useEffect(() => {
  const handleKeyPress = (e) => {
    if (e.key === '/') {
      search(query); // Sees latest query
    }
  };
  
  document.addEventListener('keypress', handleKeyPress);
  return () => document.removeEventListener('keypress', handleKeyPress);
}, [query]); // Re-attach listener when query changes
```

**But wait...** This removes and re-adds the listener on every keystroke. Inefficient!

**Better Fix:**
```javascript
// ✅ Use ref for latest value
function SearchBar() {
  const [query, setQuery] = useState('');
  const queryRef = useRef(query);
  
  useEffect(() => {
    queryRef.current = query;
  }, [query]);
  
  useEffect(() => {
    const handleKeyPress = (e) => {
      if (e.key === '/') {
        search(queryRef.current); // Always latest
      }
    };
    
    document.addEventListener('keypress', handleKeyPress);
    return () => document.removeEventListener('keypress', handleKeyPress);
  }, []); // Stable listener
  
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

## Pattern 3: Stale State in Async Operations (Race Conditions)

### The Bug:
```javascript
// ❌ Classic race condition
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetchUser(userId).then(data => {
      setUser(data); // Might be stale if userId changed!
    });
  }, [userId]);
  
  return <div>{user?.name}</div>;
}
```

**Scenario:**
1. User clicks on Alice (userId=1)
2. Slow network → fetchUser(1) takes 3 seconds
3. User clicks on Bob (userId=2)  
4. Fast network → fetchUser(2) returns in 100ms
5. setUser(Bob) → Shows Bob ✅
6. 2.9 seconds later → fetchUser(1) returns
7. setUser(Alice) → Shows Alice ❌

**Result:** UI shows Alice but URL says userId=2 (Bob). Everything breaks.

### Fix Option 1: Cleanup Flag
```javascript
// ✅ Ignore stale responses
useEffect(() => {
  let isCurrent = true;
  
  fetchUser(userId).then(data => {
    if (isCurrent) {
      setUser(data); // Only if still relevant
    }
  });
  
  return () => {
    isCurrent = false; // Mark as stale on cleanup
  };
}, [userId]);
```

### Fix Option 2: AbortController (Better)
```javascript
// ✅ Actually cancel the request
useEffect(() => {
  const controller = new AbortController();
  
  fetchUser(userId, { signal: controller.signal })
    .then(data => setUser(data))
    .catch(err => {
      if (err.name === 'AbortError') {
        // Ignored - request was cancelled
      } else {
        console.error(err);
      }
    });
  
  return () => {
    controller.abort(); // Cancel on cleanup
  };
}, [userId]);
```

### Fix Option 3: React Query (Best)
```javascript
// ✅ Handles all of this for you
function UserProfile({ userId }) {
  const { data: user } = useQuery(['user', userId], () => fetchUser(userId));
  
  return <div>{user?.name}</div>;
}

// React Query automatically:
// - Cancels stale requests
// - Caches responses
// - Handles race conditions
// - Deduplicates requests
```

## Pattern 4: Stale Props in Callbacks

### The Bug:
```javascript
// ❌ Callback passed to child has stale closure
function TodoList({ onComplete }) {
  const [todos, setTodos] = useState([]);
  
  const handleComplete = useCallback((id) => {
    const todo = todos.find(t => t.id === id); // Stale todos!
    onComplete(todo);
  }, []); // Missing dependency
  
  return <TodoItem onComplete={handleComplete} />;
}
```

### The Fix:
```javascript
// ✅ Include dependencies
const handleComplete = useCallback((id) => {
  const todo = todos.find(t => t.id === id);
  onComplete(todo);
}, [todos, onComplete]); // Will cause re-renders though...

// ✅ Better: Don't close over todos
const handleComplete = useCallback((id) => {
  setTodos(prev => {
    const todo = prev.find(t => t.id === id);
    onComplete(todo);
    return prev;
  });
}, [onComplete]); // Stable if onComplete is stable
```

## The Debugging Process

When you suspect stale state:

### Step 1: Add Logging
```javascript
useEffect(() => {
  console.log('Effect ran with:', { query, count, user });
  
  const handler = () => {
    console.log('Handler sees:', { query, count, user });
  };
  
  // Compare the two logs
}, []);
```

**If the second log shows old values:** You have a closure issue.

### Step 2: Check Dependencies
```javascript
// Install ESLint rule
// eslint-plugin-react-hooks

// It will warn you:
useEffect(() => {
  doSomething(value);
}, []); 
// ⚠️ React Hook useEffect has a missing dependency: 'value'
```

**Listen to the linter.** 99% of the time it's right.

### Step 3: Test with Rapid Changes
```javascript
// In your dev console:
for (let i = 0; i < 10; i++) {
  setTimeout(() => setUserId(i), i * 100);
}

// If your UI shows the wrong user, you have a race condition
```

## The Quick Fixes Reference

| Scenario | Problem | Solution |
|----------|---------|----------|
| **Reading state in interval/timeout** | Closure captures old state | Use `useRef` to track latest value |
| **Updating state in interval** | Always updates to same value | Use functional setState: `prev => prev + 1` |
| **Event listener reads state** | Handler has stale closure | Use `useRef` or re-attach listener with deps |
| **Async request returns late** | Race condition sets stale data | Use cleanup flag or `AbortController` |
| **useCallback captures old state** | Callback has stale closure | Add deps or use functional setState |
| **Props change but effect doesn't run** | Missing dependency | Add to dependency array |

## The Prevention Checklist

- [ ] **Every useEffect dependency is listed** (trust the ESLint warning)
- [ ] **State updates use functional form** when reading previous state
- [ ] **Refs used for latest values** in stable closures (intervals, event listeners)
- [ ] **Async operations have cleanup** (abort or ignore stale results)
- [ ] **React Query used for data fetching** (handles race conditions automatically)
- [ ] **Test rapid state changes** during development
- [ ] **Check that timers/listeners are cleaned up** on unmount

## The Golden Rules

1. **If useEffect reads state, that state should be in the dependency array**
2. **If you can't add it to deps, use a ref**
3. **For async operations, always clean up**
4. **Functional setState is your friend**: `setState(prev => ...)`
5. **When in doubt, run ESLint and fix the warnings**

## The Real-World Example

Here's all the patterns combined in one realistic component:

```javascript
function LiveChat({ roomId, userId }) {
  const [messages, setMessages] = useState([]);
  const [typing, setTyping] = useState(false);
  const typingTimeout = useRef(null);
  
  // Pattern 3: Fetch messages (race condition handling)
  useEffect(() => {
    const controller = new AbortController();
    
    fetchMessages(roomId, { signal: controller.signal })
      .then(data => setMessages(data))
      .catch(err => {
        if (err.name !== 'AbortError') console.error(err);
      });
    
    return () => controller.abort();
  }, [roomId]);
  
  // Pattern 1: WebSocket with stable connection
  useEffect(() => {
    const ws = new WebSocket(`ws://api.com/${roomId}`);
    
    ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      setMessages(prev => [...prev, msg]); // Functional update
    };
    
    return () => ws.close();
  }, [roomId]);
  
  // Pattern 1: Auto-save typing indicator
  useEffect(() => {
    if (typing) {
      const timeout = setTimeout(() => {
        setTyping(false); // Functional update not needed here
      }, 3000);
      
      typingTimeout.current = timeout;
      return () => clearTimeout(timeout);
    }
  }, [typing]);
  
  const handleType = useCallback(() => {
    setTyping(true);
    
    // Clear old timeout using ref
    if (typingTimeout.current) {
      clearTimeout(typingTimeout.current);
    }
  }, []);
  
  return (
    <div>
      <Messages messages={messages} />
      <Input onType={handleType} />
    </div>
  );
}
```

**No stale state.** All closures handled correctly.

## The Bottom Line

Stale state happens when **closures capture old values**.

**The fix is always one of:**
1. Add the value to dependencies → Effect re-runs
2. Use a ref → Always read latest value
3. Use functional setState → React gives you latest
4. Clean up async operations → Ignore stale results

Stop fighting closures. Learn the patterns. Your bugs will disappear.

---

### Further Reading
- [useEffect Dependencies](https://react.dev/learn/synchronizing-with-effects#all-variables-declared-in-the-component-body-are-reactive) - React docs
- [Closures in JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures) - MDN
- [A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/) by Dan Abramov
- [useRef for Latest Value Pattern](https://epicreact.dev/the-latest-ref-pattern-in-react/) by Kent C. Dodds