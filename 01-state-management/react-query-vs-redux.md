# React Query vs Redux: You're Comparing the Wrong Things

**Hot take:** 90% of apps using Redux don't actually need it. They're just using it as a bad cache.

## The Core Misunderstanding

React Query and Redux **solve different problems**. Comparing them is like comparing a refrigerator to a filing cabinet.

- **React Query** = Server state management (caching, synchronization, background updates)
- **Redux** = Client state management (complex state logic, predictable updates, time-travel)

**The problem?** Most apps have been using Redux for server state. That's why React Query feels like a replacement.

## The Acid Test

Look at your Redux store right now. For each piece of state, ask:

**"Did this come from an API call?"**

- Yes → You're using Redux as a cache. Use React Query instead.
- No → You might actually need Redux.

Let me show you what I mean.

## The Redux Anti-Pattern

**What 90% of Redux code looks like:**

```javascript
// ❌ Redux being used as a poor man's cache
const initialState = {
  users: [],
  loading: false,
  error: null,
  lastFetched: null
};

function usersReducer(state = initialState, action) {
  switch (action.type) {
    case 'FETCH_USERS_REQUEST':
      return { ...state, loading: true };
    case 'FETCH_USERS_SUCCESS':
      return { 
        ...state, 
        loading: false, 
        users: action.payload,
        lastFetched: Date.now() 
      };
    case 'FETCH_USERS_FAILURE':
      return { ...state, loading: false, error: action.error };
    default:
      return state;
  }
}

// Action creator with manual cache logic
function fetchUsers() {
  return async (dispatch, getState) => {
    const { lastFetched } = getState().users;
    
    // Manual stale-while-revalidate logic
    if (lastFetched && Date.now() - lastFetched < 60000) {
      return; // Cache hit
    }
    
    dispatch({ type: 'FETCH_USERS_REQUEST' });
    try {
      const response = await fetch('/api/users');
      const data = await response.json();
      dispatch({ type: 'FETCH_USERS_SUCCESS', payload: data });
    } catch (error) {
      dispatch({ type: 'FETCH_USERS_FAILURE', error });
    }
  };
}
```

**You just wrote 40+ lines to poorly replicate what React Query does in 3.**

## The React Query Way

```javascript
// ✅ React Query handling all the complexity
function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
    staleTime: 60000
  });
}

// Usage
function UserList() {
  const { data, isLoading, error } = useUsers();
  
  if (isLoading) return <Spinner />;
  if (error) return <Error error={error} />;
  return <div>{data.map(user => ...)}</div>;
}
```

**What you get for free:**
- Automatic caching
- Background refetching
- Stale-while-revalidate
- Request deduplication
- Retry logic
- Garbage collection
- Window focus refetching
- Optimistic updates
- Pagination/infinite scroll helpers

**What you had to build yourself in Redux:**
- All of the above

## When You Actually Need Redux

Redux shines when you have **complex client-side state** with **intricate update logic**.

### Good Use Case: Multi-step Wizard with Branching Logic

```javascript
// ✅ Redux making complex state manageable
const initialState = {
  step: 1,
  formData: {
    accountType: null,
    personalInfo: {},
    businessInfo: {},
    paymentMethod: null
  },
  validationErrors: {},
  completedSteps: []
};

function wizardReducer(state, action) {
  switch (action.type) {
    case 'NEXT_STEP':
      // Complex logic: skip steps based on account type
      const nextStep = calculateNextStep(state);
      return {
        ...state,
        step: nextStep,
        completedSteps: [...state.completedSteps, state.step]
      };
      
    case 'UPDATE_ACCOUNT_TYPE':
      // Changing account type clears irrelevant data
      return {
        ...state,
        formData: {
          ...state.formData,
          accountType: action.payload,
          businessInfo: action.payload === 'personal' ? {} : state.formData.businessInfo
        }
      };
      
    case 'JUMP_TO_STEP':
      // Can only jump to completed steps
      if (!state.completedSteps.includes(action.step)) {
        return state;
      }
      return { ...state, step: action.step };
      
    default:
      return state;
  }
}
```

**Why Redux here?**
- Complex state transitions
- Multiple places need to trigger state updates
- You want a single source of truth for the wizard state
- The state is NOT fetched from a server

### Good Use Case: Real-time Collaborative Editing

```javascript
// ✅ Redux managing complex client state + WebSocket updates
const editorState = {
  document: { title: '', content: [] },
  cursors: {}, // Other users' cursor positions
  pendingChanges: [],
  conflictResolution: 'operational-transform',
  localChanges: [],
  syncStatus: 'synced'
};

// Actions from WebSocket, local edits, conflict resolution
// This is complex client logic, not simple data fetching
```

## The Hybrid Approach (Most Apps)

**Reality:** Most apps need React Query + minimal client state (useState/useReducer).

```javascript
// Server state: React Query
function TodoApp() {
  const { data: todos } = useQuery(['todos'], fetchTodos);
  const updateTodo = useMutation(updateTodoApi, {
    onMutate: async (newTodo) => {
      // Optimistic update
      await queryClient.cancelQueries(['todos']);
      const previous = queryClient.getQueryData(['todos']);
      queryClient.setQueryData(['todos'], old => 
        old.map(t => t.id === newTodo.id ? newTodo : t)
      );
      return { previous };
    },
    onError: (err, newTodo, context) => {
      // Rollback on error
      queryClient.setQueryData(['todos'], context.previous);
    }
  });
  
  // Client state: Local useState
  const [filter, setFilter] = useState('all');
  const [selectedIds, setSelectedIds] = useState([]);
  
  // Derived state: Computed
  const filteredTodos = useMemo(
    () => todos?.filter(t => filter === 'all' || t.status === filter),
    [todos, filter]
  );
  
  return <TodoList todos={filteredTodos} />;
}
```

**No Redux needed.** Server state is cached and synced. Client state is local. Everything works.

## The Migration Decision

**Currently using Redux for API data?** Here's your ROI calculation:

### What you remove:
- Action types (FETCH_X_REQUEST, FETCH_X_SUCCESS, FETCH_X_FAILURE)
- Action creators with try/catch blocks
- Reducers for loading/error states
- Manual cache invalidation logic
- Selectors for derived data
- Middleware for async logic
- 60-80% of your Redux code

### What you add:
- `npm install @tanstack/react-query`
- Query definitions (simpler than what you had)
- Maybe 20% of the original lines of code

### What you get:
- Better UX (automatic background updates, instant cached responses)
- Less bugs (no stale data, no manual cache management)
- Smaller bundle (remove Redux + middleware)
- Faster dev velocity (write features, not boilerplate)

## The Reality Check

**"But we have so much Redux code already!"**

Migrate incrementally:

1. **Week 1:** New features use React Query
2. **Week 2-4:** Migrate read-heavy routes (dashboards, lists)
3. **Month 2:** Migrate write-heavy routes (forms, mutations)
4. **Month 3:** Delete Redux if nothing's left

You don't rewrite. You gradually replace.

**"What about Redux DevTools?"**

React Query has its own DevTools. They're excellent for inspecting cache state, query status, and refetch behavior.

**"What about middleware like Redux-Saga?"**

If you're using Redux-Saga to orchestrate API calls... that's exactly what React Query does. You're maintaining complex machinery to do what React Query does out of the box.

## The Comparison Table

| Concern | Redux | React Query |
|---------|-------|-------------|
| **API data caching** | Manual, error-prone | Automatic, robust |
| **Loading states** | You build it | Built-in |
| **Error handling** | You build it | Built-in |
| **Refetching** | You build it | Built-in |
| **Optimistic updates** | Complex | 10 lines |
| **Request deduplication** | You build it | Built-in |
| **Cache invalidation** | Manual, fragile | Automatic + manual options |
| **Stale data** | Your problem | Handled automatically |
| **Bundle size** | ~10kb (+ middleware) | ~12kb |
| **Boilerplate** | High | Minimal |
| **DevTools** | Excellent | Excellent |
| **Learning curve** | Steep | Gentle |
| **Best for** | Complex client state | Server state |

## The Decision Framework

```
Need to manage data from an API?
├─ Yes → React Query
└─ No → Is the client state complex?
    ├─ Yes (multi-step flows, collaborative features, complex sync) → Redux/Zustand
    └─ No → useState/useReducer
```

**In practice:**
- 90% of your "state" → React Query
- 8% of your "state" → Local useState/useReducer  
- 2% of your "state" → Global client state (Redux/Zustand)

## The Bottom Line

**Redux is not dead.** It's just overused.

If your Redux store is mostly API data, you're using the wrong tool. React Query isn't a Redux competitor—it's a wake-up call that most apps never needed Redux in the first place.

**Use Redux for:** Complex client state with intricate update logic
**Use React Query for:** Everything that comes from a server

Stop building caches. Start building features.

---

### Further Reading
- [React Query vs Redux](https://tkdodo.eu/blog/react-query-and-state-management) by TkDodo
- [Practical React Query](https://tkdodo.eu/blog/practical-react-query) by TkDodo  
- [You Might Not Need Redux](https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367) by Dan Abramov
- [Redux vs React Query](https://blog.logrocket.com/redux-vs-react-query/) on LogRocket