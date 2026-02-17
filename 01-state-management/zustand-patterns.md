# Zustand Patterns

Zustand has quietly become one of the most pragmatic state management choices in the React ecosystem. It avoids the boilerplate of Redux, sidesteps the re-render pitfalls of Context, and stays out of your way. But "simple" doesn't mean "without nuance" — knowing the right patterns separates a maintainable store from a tangled one.

---

## Why Zustand Works

Zustand's mental model is refreshingly direct: a store is a hook. You define state and actions together, subscribe to exactly what you need, and React only re-renders the components that care. Under the hood it uses a pub/sub mechanism outside of React's tree, which means no Provider wrapping and no context overhead.

```ts
import { create } from 'zustand'

interface BearStore {
  count: number
  increment: () => void
  reset: () => void
}

export const useBearStore = create<BearStore>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  reset: () => set({ count: 0 }),
}))
```

That's a complete, production-ready store. No actions file, no reducer, no dispatch.

---

## Pattern 1: Atomic Selectors

The single biggest mistake with Zustand is subscribing to the entire store object.

```ts
// ❌ Re-renders on ANY store change
const store = useBearStore()

// ✅ Re-renders only when `count` changes
const count = useBearStore((state) => state.count)
```

When you select a primitive value, Zustand does a strict equality check (`===`) before deciding to re-render. Selecting the whole store object always fails that check because the reference changes on every update.

For multiple values, prefer separate selectors over destructuring an object selector:

```ts
// ❌ New object reference on every render
const { count, status } = useBearStore((state) => ({
  count: state.count,
  status: state.status,
}))

// ✅ Two independent subscriptions
const count = useBearStore((state) => state.count)
const status = useBearStore((state) => state.status)
```

If you genuinely need multiple values in one selector, pass `shallow` as the equality function:

```ts
import { shallow } from 'zustand/shallow'

const { count, status } = useBearStore(
  (state) => ({ count: state.count, status: state.status }),
  shallow
)
```

---

## Pattern 2: Colocate Actions with State

Zustand lets you define actions directly inside the store. Take advantage of this — it keeps behavior close to the data it operates on and makes your components thinner.

```ts
interface CartStore {
  items: CartItem[]
  total: number
  addItem: (item: CartItem) => void
  removeItem: (id: string) => void
  clearCart: () => void
}

export const useCartStore = create<CartStore>((set, get) => ({
  items: [],
  total: 0,

  addItem: (item) =>
    set((state) => {
      const items = [...state.items, item]
      return { items, total: items.reduce((sum, i) => sum + i.price, 0) }
    }),

  removeItem: (id) =>
    set((state) => {
      const items = state.items.filter((i) => i.id !== id)
      return { items, total: items.reduce((sum, i) => sum + i.price, 0) }
    }),

  clearCart: () => set({ items: [], total: 0 }),
}))
```

Notice the use of `get` — it's available alongside `set` and lets you read current state inside actions without needing to close over stale values. This is especially useful for actions that are called outside of React components (e.g., in utility functions or event handlers).

---

## Pattern 3: Slice Pattern for Large Stores

As your app grows, a single monolithic store becomes hard to navigate. The slice pattern splits your store into logical sections while keeping a single store instance.

```ts
// slices/authSlice.ts
export interface AuthSlice {
  user: User | null
  isAuthenticated: boolean
  login: (user: User) => void
  logout: () => void
}

export const createAuthSlice = (set: SetState<AppStore>): AuthSlice => ({
  user: null,
  isAuthenticated: false,
  login: (user) => set({ user, isAuthenticated: true }),
  logout: () => set({ user: null, isAuthenticated: false }),
})

// slices/uiSlice.ts
export interface UISlice {
  theme: 'light' | 'dark'
  sidebarOpen: boolean
  toggleTheme: () => void
  toggleSidebar: () => void
}

export const createUISlice = (set: SetState<AppStore>): UISlice => ({
  theme: 'light',
  sidebarOpen: false,
  toggleTheme: () =>
    set((state) => ({ theme: state.theme === 'light' ? 'dark' : 'light' })),
  toggleSidebar: () =>
    set((state) => ({ sidebarOpen: !state.sidebarOpen })),
})

// store.ts
type AppStore = AuthSlice & UISlice

export const useAppStore = create<AppStore>()((...args) => ({
  ...createAuthSlice(...args),
  ...createUISlice(...args),
}))
```

Each slice is independently testable, and you can even keep them in separate files while still sharing a single store subscription mechanism.

---

## Pattern 4: Middleware Composition

Zustand has a clean middleware system. The most commonly useful ones are `persist`, `devtools`, and `immer`.

### Persist

```ts
import { persist, createJSONStorage } from 'zustand/middleware'

export const useSettingsStore = create<SettingsStore>()(
  persist(
    (set) => ({
      theme: 'light',
      language: 'en',
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
    }),
    {
      name: 'user-settings',
      storage: createJSONStorage(() => localStorage),
      // Only persist specific keys
      partialize: (state) => ({ theme: state.theme, language: state.language }),
    }
  )
)
```

### Devtools + Immer

```ts
import { devtools } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

export const useComplexStore = create<ComplexStore>()(
  devtools(
    immer((set) => ({
      nested: { deeply: { value: 0 } },
      updateDeep: () =>
        set((state) => {
          // Immer lets you mutate directly
          state.nested.deeply.value += 1
        }),
    })),
    { name: 'ComplexStore' }
  )
)
```

When stacking middleware, the order matters: `devtools` should generally be outermost so it can observe all state changes, while `immer` should wrap your store definition directly.

---

## Pattern 5: Derived State with Selectors

Avoid storing derived values in your store. Instead, compute them in selectors — either inline or as named functions you can reuse and test independently.

```ts
// ❌ Storing derived state
interface TodoStore {
  todos: Todo[]
  completedCount: number // redundant
  pendingCount: number   // redundant
}

// ✅ Compute in selectors
const completedCount = useTodoStore(
  (state) => state.todos.filter((t) => t.done).length
)

// For expensive computations, memoize with a library like reselect
// or co-locate the memoization with your component using useMemo
const expensiveStats = useMemo(
  () => computeStats(todos),
  [todos]
)
```

If a selector is used in many places and the computation is expensive, extract it:

```ts
// selectors/todoSelectors.ts
export const selectCompletedTodos = (state: TodoStore) =>
  state.todos.filter((t) => t.done)

export const selectOverdueTodos = (state: TodoStore) =>
  state.todos.filter((t) => !t.done && isPast(t.dueDate))
```

---

## Pattern 6: Async Actions

Zustand handles async naturally — actions are just functions, and `set` can be called after any await.

```ts
interface UserStore {
  users: User[]
  loading: boolean
  error: string | null
  fetchUsers: () => Promise<void>
}

export const useUserStore = create<UserStore>((set) => ({
  users: [],
  loading: false,
  error: null,

  fetchUsers: async () => {
    set({ loading: true, error: null })
    try {
      const users = await api.getUsers()
      set({ users, loading: false })
    } catch (err) {
      set({ error: (err as Error).message, loading: false })
    }
  },
}))
```

One thing to be aware of: if a component unmounts between the `await` and the final `set`, Zustand will still update the store (it doesn't throw), but components that have re-mounted or other subscribers may see the update. This is usually fine, and preferable to the cleanup ceremony required by `useEffect`.

For server state specifically — data you fetch, cache, and synchronize — consider whether Zustand is the right tool at all. React Query or SWR handle cache invalidation, background refetching, and deduplication better than a hand-rolled solution in Zustand.

---

## Pattern 7: Zustand Outside React

One of Zustand's underrated features is that the store exists outside the React tree. You can read and write state from anywhere.

```ts
// In an event listener, WebSocket handler, or utility function
import { useAuthStore } from '@/store/authStore'

wsClient.on('unauthorized', () => {
  useAuthStore.getState().logout()
})

// Read state without subscribing
const isAuthenticated = useAuthStore.getState().isAuthenticated

// Subscribe manually
const unsubscribe = useAuthStore.subscribe(
  (state) => state.user,
  (user) => console.log('User changed:', user)
)
```

This makes Zustand a solid choice for managing state that needs to be accessible in non-component contexts like API interceptors, analytics, or background sync workers.

---

## When Not to Use Zustand

Zustand is not always the right answer. Keep it in its lane:

**Use local state (`useState`) when** the state is truly local to a component or a small subtree, doesn't need to be shared, and its lifecycle matches the component's lifecycle.

**Use React Query or SWR when** you're dealing with server state — data that lives on a remote server and needs caching, synchronization, and refetching logic.

**Use URL state when** the state should be bookmarkable or shareable (search filters, pagination, selected tabs).

Zustand shines for client-side application state that is genuinely shared across the tree: UI state like theme and modals, user session data, shopping carts, and editor state.

---

## Checklist

Before shipping a Zustand store, run through this:

- [ ] Selectors are atomic — components subscribe to primitives, not objects
- [ ] `shallow` is used where object selectors are unavoidable
- [ ] Derived values are computed in selectors, not stored
- [ ] Actions live inside the store definition, not in components
- [ ] Async actions handle loading and error states
- [ ] Large stores use the slice pattern
- [ ] `persist` uses `partialize` to exclude transient state
- [ ] `devtools` middleware is added for debugging in development