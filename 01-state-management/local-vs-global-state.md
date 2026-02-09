# Local vs Global State: The Real Decision Framework

Most articles tell you *what* local and global state are. This one tells you *when* to use each and *why* it matters.

## The Core Principle

**State should live at the lowest level possible, and only be promoted when you have evidence it needs to be shared.**

Not "might need to be shared someday." Not "could be useful elsewhere." **Evidence.**

## What Actually Qualifies as Global State?

After shipping dozens of production apps, here's what I've found genuinely belongs in global state:

### 1. Authentication State
```javascript
// Global: YES
const auth = {
  user: { id, name, email, permissions },
  token: "...",
  isAuthenticated: true
}
```
**Why?** Every route, every component that makes an API call needs this. It changes infrequently but is read constantly.

### 2. Theme/User Preferences
```javascript
// Global: YES
const preferences = {
  theme: 'dark',
  language: 'en',
  timezone: 'America/New_York',
  accessibility: { reducedMotion: true }
}
```
**Why?** UI-wide concern. You don't want to prop-drill theme 12 levels deep.

### 3. Real-time Connection State
```javascript
// Global: YES
const connection = {
  websocket: WebSocket,
  status: 'connected',
  lastPing: timestamp,
  reconnectAttempts: 0
}
```
**Why?** Multiple features need to know: chat, notifications, live updates. It's infrastructure.

### What People Think is Global (But Isn't)

#### ❌ "Current selected item"
```javascript
// DON'T do this
const globalState = {
  selectedUserId: 123,
  selectedProductId: 456,
  selectedTabIndex: 2
}
```

This is **local UI state**. It belongs to the component/page that owns that selection.

**The test:** If you navigate away and come back, should the selection persist? 
- No? → Local state
- Yes? → URL parameter or local storage, still not global state

#### ❌ "List of items for this page"
```javascript
// DON'T do this
const globalState = {
  dashboardUsers: [...],
  settingsUsers: [...],
  profileUsers: [...]
}
```

Use a caching solution (React Query, SWR) or keep it local. Global state isn't a database.

#### ❌ "Form data"
```javascript
// DON'T do this
const globalState = {
  loginForm: { email: '', password: '' },
  signupForm: { email: '', password: '', name: '' }
}
```

Forms are local. Even multi-step forms. Use local state + URL parameters for steps.

**Exception:** Draft auto-save for critical forms (job applications, tax returns) → localStorage, not global state.

## The Decision Tree

```
Does more than one route need it?
├─ No → Local state
└─ Yes
    │
    Is it derived from URL/props?
    ├─ Yes → Not state at all, compute it
    └─ No
        │
        Does it change frequently (>10 times/minute)?
        ├─ Yes → Local state + sync mechanism (WebSocket, polling)
        └─ No
            │
            Is it user-specific and persisted?
            ├─ Yes → Global state + server sync
            └─ No → Probably local, use context if needed
```

## The Real-World Scenarios

### Scenario 1: Shopping Cart

**Junior approach:**
```javascript
// ❌ Everything in Redux
const state = {
  cart: {
    items: [...],
    isOpen: false,  // ← This is local!
    selectedItem: null,  // ← This is local!
    checkoutStep: 1  // ← This is local!
  }
}
```

**Better approach:**
```javascript
// Global: Just the cart data
const globalState = {
  cartItems: [...]  // This needs to persist, show in header, etc.
}

// Local: UI state in CartDrawer component
function CartDrawer() {
  const [isOpen, setIsOpen] = useState(false);
  const [editingItem, setEditingItem] = useState(null);
  const cartItems = useCartItems(); // From global
  
  // ...
}
```

### Scenario 2: Notification System

**The mistake:**
```javascript
// ❌ Storing notifications in global state
const state = {
  notifications: [
    { id: 1, message: "Welcome!", read: false },
    { id: 2, message: "New message", read: false }
  ]
}
```

**Why it's wrong:**
- Notifications should come from the server
- They're not state you're managing, they're data you're fetching
- Use React Query/SWR with polling or WebSocket updates

**Better:**
```javascript
// Global: Connection to notification service
const notificationService = {
  connection: WebSocket,
  subscribe: (callback) => {},
  markAsRead: (id) => {}
}

// In component: Fetch and cache
function Notifications() {
  const { data: notifications } = useQuery(
    'notifications',
    fetchNotifications,
    { refetchInterval: 30000 }
  );
  
  // ...
}
```

### Scenario 3: Modal Management

**Common trap:**
```javascript
// ❌ Modal state in Redux
const state = {
  modals: {
    loginModal: { isOpen: true, data: {...} },
    confirmModal: { isOpen: false, data: null },
    imageViewerModal: { isOpen: false, imageUrl: '' }
  }
}
```

**Reality check:** When's the last time you needed to know if a modal was open from a completely different part of your app?

**Better pattern:**
```javascript
// Local state where the trigger lives
function UserProfile() {
  const [showDeleteConfirm, setShowDeleteConfirm] = useState(false);
  
  return (
    <>
      <button onClick={() => setShowDeleteConfirm(true)}>Delete</button>
      {showDeleteConfirm && (
        <ConfirmModal
          onConfirm={handleDelete}
          onCancel={() => setShowDeleteConfirm(false)}
        />
      )}
    </>
  );
}
```

Or use a modal service with React Context if you want programmatic control:
```javascript
const { showModal } = useModals();

function handleDelete() {
  showModal({
    type: 'confirm',
    title: 'Delete user?',
    onConfirm: () => deleteUser()
  });
}
```

## The Performance Angle

### Global State Re-render Cost

Every time global state changes, **every component that subscribes to it re-evaluates**.

```javascript
// ❌ One giant state object
const state = {
  user: {...},
  theme: {...},
  cart: {...},
  notifications: {...}
}

// Component only needs user, but re-renders when cart changes
function UserProfile() {
  const { user } = useGlobalState(); // Re-renders on ANY state change
  return <div>{user.name}</div>;
}
```

**Solution 1: Slice your state**
```javascript
// ✅ Separate stores/slices
const user = useUserState();  // Only re-renders when user changes
const theme = useTheme();     // Only re-renders when theme changes
```

**Solution 2: Selectors with shallow equality**
```javascript
// ✅ Select only what you need
const userName = useGlobalState(state => state.user.name);
// Only re-renders when user.name changes, not the whole user object
```

### Local State is Free

```javascript
function TodoItem({ todo }) {
  const [isEditing, setIsEditing] = useState(false);
  const [showMenu, setShowMenu] = useState(false);
  
  // These only cause THIS component to re-render
  // Zero cost to the rest of your app
}
```

**The guideline:** If state only affects one component and its direct children, it costs nothing to keep it local. Make it global and you're paying a performance tax.

## The Maintenance Angle

### Global State is a Commitment

When you put something in global state, you're saying:
1. This is important enough to be centrally managed
2. I'm willing to write actions/reducers/mutations for it
3. I'll maintain its shape as a contract across the app
4. I'll handle its persistence, hydration, and sync

**Ask yourself:** Is `isDropdownOpen` worth that commitment? Probably not.

### Local State is Disposable

```javascript
function FilterPanel() {
  const [filters, setFilters] = useState({});
  const [isExpanded, setIsExpanded] = useState(false);
  const [searchQuery, setSearchQuery] = useState('');
  
  // Component unmounts? State is gone. Zero cleanup needed.
  // Refactor the component? State moves with it.
  // Delete the feature? State disappears automatically.
}
```

This is a **feature**, not a bug.

## The Anti-Patterns

### 1. "Let's just put it in Redux to be safe"

**Why it's wrong:** You're trading simplicity for imaginary future-proofing. YAGNI applies to state management.

**Evidence I'd accept:**
- "We show this in the header and on the settings page" ✅
- "We might need it somewhere later" ❌

### 2. "We need global state for prop drilling"

**Wrong tool.** Prop drilling isn't solved by global state, it's solved by:
- Component composition
- React Context
- Better component architecture

**Example:**
```javascript
// ❌ Redux for prop drilling
function App() {
  const theme = useSelector(state => state.theme);
  return <Page theme={theme} />
}

function Page({ theme }) {
  return <Section theme={theme} />
}

function Section({ theme }) {
  return <Button theme={theme} />
}

// ✅ Context for app-wide concerns
function App() {
  return (
    <ThemeProvider value="dark">
      <Page />
    </ThemeProvider>
  );
}

function Button() {
  const theme = useTheme(); // No drilling needed
}
```

### 3. "Lifting state to the nearest common ancestor"

This advice is outdated. Modern tools:

```javascript
// Old way: Lift to common ancestor
function Dashboard() {
  const [selectedUserId, setSelectedUserId] = useState(null);
  
  return (
    <>
      <UserList onSelect={setSelectedUserId} />
      <UserDetails userId={selectedUserId} />
    </>
  );
}

// Modern way: Use URL state
function Dashboard() {
  return (
    <>
      <UserList />
      <UserDetails />
    </>
  );
}

function UserList() {
  const navigate = useNavigate();
  return <div onClick={() => navigate(`/user/${id}`)} />;
}

function UserDetails() {
  const { userId } = useParams();
  const { data } = useQuery(['user', userId], fetchUser);
  return <div>{data.name}</div>;
}
```

URL is your state. It's global, it's shareable, it's bookmarkable.

## The Hybrid Approach: URL + Server + Local

**Modern apps don't need much global state at all.**

- **URL** = Navigation state, filters, selected items, tabs
- **Server** (React Query/SWR) = Data, caching, optimistic updates  
- **Local** = UI state, form inputs, toggles, modals
- **Global** = Auth, theme, real-time connection, truly app-wide state

**Example: A dashboard with filters**

```javascript
// NO global state needed
function Dashboard() {
  // URL manages filter state
  const [searchParams] = useSearchParams();
  const status = searchParams.get('status') || 'all';
  const sort = searchParams.get('sort') || 'recent';
  
  // Server state managed by React Query
  const { data } = useQuery(
    ['tickets', status, sort],
    () => fetchTickets({ status, sort })
  );
  
  // Local UI state
  const [selectedIds, setSelectedIds] = useState([]);
  const [viewMode, setViewMode] = useState('grid');
  
  return <TicketGrid tickets={data} />;
}
```

Everything is managed, nothing is global. The state is where it should be.

## The Migration Path

**Already have too much global state?** Here's how to fix it:

### Step 1: Identify True Global State
Run this audit:
```
For each piece of global state, ask:
1. Is it used in 2+ routes? (Yes → Keep, No → Move to local)
2. Is it derived from URL/props? (Yes → Delete, compute it)
3. Is it really data we should fetch? (Yes → Use React Query)
4. Is it UI state? (Yes → Move to local or URL)
```

### Step 2: Extract UI State First
Easiest wins. Modal state, dropdown state, form validation state → move to components.

### Step 3: Move to URL State
Anything that should survive refresh or be shareable → URL parameters.

### Step 4: Migrate to Server State
Lists, user data, anything fetched from API → React Query/SWR.

### Step 5: Keep What's Left
You'll likely end up with 5-10 pieces of true global state. That's healthy.

## The Mental Model

Think of your app as concentric circles:

```
┌─────────────────────────────────────┐
│  URL State (global, bookmarkable)   │
│  ┌───────────────────────────────┐  │
│  │  Global State (auth, theme)   │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │  Server State (cached)  │  │  │
│  │  │  ┌───────────────────┐  │  │  │
│  │  │  │  Local State      │  │  │  │
│  │  │  │  (UI, forms)      │  │  │  │
│  │  │  └───────────────────┘  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

State flows inward → Local state is the most specific, global state is the most broad.

## The Checklist

Before adding to global state, ask:

- [ ] Is this used in 2+ routes? (Not "might be", **is**)
- [ ] Have I tried URL parameters?
- [ ] Have I tried React Query/SWR?
- [ ] Have I tried Context for this subtree?
- [ ] Is this *really* not just UI state?
- [ ] Am I adding this for a concrete need or "just in case"?

If you can't check most of these boxes, it doesn't belong in global state.

## The Bottom Line

**Local by default. Global by necessity.**

Your future self (and your teammates) will thank you for the restraint.

---

### Further Reading
- [Application State Management with React](https://kentcdodds.com/blog/application-state-management-with-react) by Kent C. Dodds
- [You Might Not Need Redux](https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367) by Dan Abramov
- [React Query: It's Time to Break up with your "Global State"](https://tkdodo.eu/blog/react-query-and-forms) by TkDodo