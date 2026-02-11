# Duplicate Messages: The Bug That Haunts Every Real-Time App

**The symptom:** Messages appear twice (or three times, or ten times) in your chat/notification/feed.

**The feeling:** "It worked fine in development..."

**The reality:** You have one of five classic bugs. Let me show you how to find which one.

## The Debug Process

When you see duplicate messages, **don't start fixing randomly**. Follow this:

### Step 1: Where's the Duplication?

Open your browser DevTools Network tab. Filter by WebSocket or fetch requests.

**Question:** Do you see duplicate API calls/WebSocket messages?

- **Yes** → The bug is in your event handlers/subscriptions (Jump to Pattern 1-3)
- **No** → The bug is in your React rendering (Jump to Pattern 4-5)

This ONE check saves you hours.

## Pattern 1: Double Event Listeners (Most Common)

**The Bug:**
```javascript
// ❌ This creates a NEW listener on every render
function ChatRoom({ roomId }) {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    // Subscribes on every render where roomId changes
    socket.on('message', (msg) => {
      setMessages(prev => [...prev, msg]);
    });
    
    // Missing cleanup!
  }, [roomId]);
  
  return <MessageList messages={messages} />;
}
```

**What happens:**
1. Component mounts → Subscribe to 'message' event
2. User switches rooms (roomId changes) → Effect runs again
3. Now you have TWO listeners for 'message'
4. New message arrives → Both listeners fire → Duplicate message

**The Network Tab:** Shows one WebSocket message, but it triggers twice in your app.

**The Fix:**
```javascript
// ✅ Cleanup removes old listener before adding new one
function ChatRoom({ roomId }) {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    const handleMessage = (msg) => {
      setMessages(prev => [...prev, msg]);
    };
    
    socket.on('message', handleMessage);
    
    return () => {
      socket.off('message', handleMessage); // CRITICAL
    };
  }, [roomId]);
  
  return <MessageList messages={messages} />;
}
```

**How to verify:** Add a console.log in the listener. If you see it twice per message, you have this bug.

## Pattern 2: Strict Mode in Development

**The Symptom:** Duplicates ONLY in development, production is fine.

**The Bug:**
```javascript
// React 18+ Strict Mode mounts components twice
function Chat() {
  useEffect(() => {
    // This runs TWICE in dev (mount, unmount, mount)
    const ws = new WebSocket('ws://localhost:3000');
    
    ws.onmessage = (event) => {
      addMessage(event.data); // Called twice!
    };
    
    // No cleanup = second WebSocket stays alive
  }, []);
}
```

**What's happening:**
- Strict Mode intentionally mounts → unmounts → remounts to catch bugs
- You open WebSocket on mount
- You don't close it on unmount
- On remount, you open ANOTHER WebSocket
- Now you have TWO connections to the server

**The Network Tab:** Shows TWO WebSocket connections.

**The Fix:**
```javascript
// ✅ Cleanup closes the WebSocket
function Chat() {
  useEffect(() => {
    const ws = new WebSocket('ws://localhost:3000');
    
    ws.onmessage = (event) => {
      addMessage(event.data);
    };
    
    return () => {
      ws.close(); // CRITICAL
    };
  }, []);
}
```

**Pro tip:** If it only happens in dev, check for missing cleanup functions. Strict Mode is your friend—it's exposing a real bug.

## Pattern 3: Race Condition with Multiple Subscribers

**The Bug:**
```javascript
// ❌ Multiple components subscribing to the same event
function NotificationBadge() {
  useEffect(() => {
    socket.on('notification', (notif) => {
      // Add to global state
      dispatch(addNotification(notif));
    });
  }, []);
}

function NotificationPanel() {
  useEffect(() => {
    socket.on('notification', (notif) => {
      // Same event, different handler
      dispatch(addNotification(notif)); // DUPLICATE!
    });
  }, []);
}

// Both components are listening to the same event
// One server message → Two dispatches → Duplicate in state
```

**The Network Tab:** One WebSocket message, but multiple parts of your app react to it.

**The Fix: Centralize Subscriptions**
```javascript
// ✅ Single source of truth for WebSocket events
function WebSocketProvider({ children }) {
  useEffect(() => {
    socket.on('notification', (notif) => {
      dispatch(addNotification(notif)); // Only once
    });
    
    return () => socket.off('notification');
  }, []);
  
  return children;
}

// Components just read from state, don't subscribe
function NotificationBadge() {
  const count = useSelector(state => state.notifications.length);
  return <Badge count={count} />;
}
```

**Rule:** One event listener per event type, at the app level. Components consume state, not events.

## Pattern 4: Missing Key Prop (React Rendering Bug)

**The Bug:**
```javascript
// ❌ No key or non-unique key
function MessageList({ messages }) {
  return (
    <div>
      {messages.map((msg) => (
        <Message message={msg} /> // Missing key!
      ))}
    </div>
  );
}

// Or worse:
{messages.map((msg, index) => (
  <Message key={index} message={msg} /> // Index as key is BAD
))}
```

**What happens:**
1. Array has [msg1, msg2]
2. New message arrives: [msg1, msg2, msg3]
3. React sees key={0}, key={1}, key={2}
4. Because you used index, React thinks msg2 moved from index 1 to index 2
5. React re-renders existing components in wrong positions
6. Depending on your Message component logic, this can cause duplicates or messages appearing in wrong order

**The Network Tab:** Shows correct data, but UI is wrong.

**The Fix:**
```javascript
// ✅ Use unique, stable ID
function MessageList({ messages }) {
  return (
    <div>
      {messages.map((msg) => (
        <Message key={msg.id} message={msg} />
      ))}
    </div>
  );
}
```

**How to verify:** React DevTools → Profiler → Look for unnecessary re-renders. If Message components are mounting/unmounting when they shouldn't, it's a key issue.

## Pattern 5: State Updates in Quick Succession

**The Bug:**
```javascript
// ❌ Multiple setState calls creating duplicates
function Chat() {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    socket.on('message', (msg) => {
      // Problem: If two messages arrive quickly...
      setMessages([...messages, msg]); // Both closures see old state!
    });
  }, [messages]); // Dependency on messages causes re-subscription
}
```

**What happens:**
1. State is [msg1]
2. msg2 arrives → setMessages([msg1, msg2])
3. Before render, msg3 arrives → setMessages([msg1, msg3]) // Still sees old state!
4. Result: You lose msg2, or get weird duplicates depending on timing

**The Fix: Functional Updates**
```javascript
// ✅ Use function form of setState
function Chat() {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    const handleMessage = (msg) => {
      setMessages(prev => [...prev, msg]); // Always see latest state
    };
    
    socket.on('message', handleMessage);
    return () => socket.off('message', handleMessage);
  }, []); // No dependency on messages = stable subscription
}
```

**Key insight:** If your useEffect depends on the state it's updating, you're probably doing it wrong.

## The Production Debugging Kit

When you can't reproduce locally:

### 1. Add Unique Message IDs
```javascript
// Server-side: Add UUID to every message
{
  id: 'uuid-here',
  content: 'Hello',
  timestamp: Date.now()
}

// Client-side: Detect duplicates
const seenIds = new Set();

socket.on('message', (msg) => {
  if (seenIds.has(msg.id)) {
    console.error('DUPLICATE MESSAGE:', msg.id);
    // Log to your error tracking (Sentry, etc.)
    return; // Don't add it
  }
  seenIds.add(msg.id);
  addMessage(msg);
});
```

### 2. Count Event Listeners
```javascript
// In development, check listener count
useEffect(() => {
  console.log('Active listeners:', socket.listenerCount('message'));
  
  socket.on('message', handleMessage);
  
  return () => {
    socket.off('message', handleMessage);
    console.log('After cleanup:', socket.listenerCount('message'));
  };
}, []);

// Should always be 1 (or your expected count)
// If it grows every render, you have Pattern 1
```

### 3. Network Waterfall Analysis
```javascript
// Add request ID to API calls
fetch('/api/messages', {
  headers: { 'X-Request-ID': generateId() }
});

// Check server logs for duplicate request IDs
// If same ID appears twice = client is making duplicate calls
// If different IDs = client is correctly calling twice
```

## The Prevention Checklist

Before you ship real-time features:

- [ ] **Every event listener has a cleanup function**
- [ ] **List items have unique, stable keys** (not index)
- [ ] **State updates use functional form** when depending on previous state
- [ ] **Only ONE subscriber per event** at the app level
- [ ] **Server sends unique IDs** for deduplication
- [ ] **Test in Strict Mode** (React 18+)
- [ ] **Test rapid event bursts** (send 10 messages in 100ms)
- [ ] **Test route changes** mid-operation (listeners persist?)

## The Real Fix

Most duplicate bugs come from **missing cleanup functions**.

**The pattern:**
```javascript
useEffect(() => {
  // Setup
  const subscription = subscribe();
  
  // ALWAYS return cleanup
  return () => {
    subscription.unsubscribe();
  };
}, [deps]);
```

If your useEffect doesn't return a cleanup function, ask yourself: "What happens when this component unmounts or re-runs?"

Usually, the answer is: "Oh... I create a duplicate listener."

## Common Variations

**Duplicate API calls:**
```javascript
// ❌ Multiple invocations
useEffect(() => {
  fetchData();
}, [id]); // Runs on every id change, might overlap

// ✅ Cancel previous request
useEffect(() => {
  const controller = new AbortController();
  
  fetchData({ signal: controller.signal });
  
  return () => controller.abort(); // Cancel on cleanup
}, [id]);
```

**Duplicate form submissions:**
```javascript
// ❌ No debounce/disable
<button onClick={handleSubmit}>Submit</button>

// ✅ Prevent double-click
const [submitting, setSubmitting] = useState(false);

<button 
  onClick={handleSubmit} 
  disabled={submitting}
>
  {submitting ? 'Submitting...' : 'Submit'}
</button>
```

**Duplicate analytics events:**
```javascript
// ❌ Fires on every render
function ProductPage({ productId }) {
  analytics.track('Product Viewed', { productId }); // NO!
}

// ✅ Fire once per product
useEffect(() => {
  analytics.track('Product Viewed', { productId });
}, [productId]);
```

## The Bottom Line

**99% of duplicate message bugs are:**
1. Missing cleanup in useEffect (70%)
2. Missing/wrong keys (15%)
3. Multiple subscribers (10%)
4. Everything else (5%)

Start with #1. Check your cleanups. You'll fix most duplicates instantly.

---

### Further Reading
- [React useEffect Cleanup](https://react.dev/learn/synchronizing-with-effects#how-to-handle-the-effect-firing-twice-in-development) - Official React docs
- [WebSocket Best Practices](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket) - MDN
- [Why Keys Matter](https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key) - React docs