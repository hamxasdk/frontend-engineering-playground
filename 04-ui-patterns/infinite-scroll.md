# Infinite Scroll: Beyond the Tutorial

**The tutorial version:** "Just load more when you scroll to the bottom!"

**The production version:** Race conditions, memory leaks, scroll position bugs, duplicate requests, and users who scroll REALLY fast.

Let me show you how to build it right.

## The Core Pattern

```javascript
// ✅ Production-ready infinite scroll
function InfiniteList() {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const observerTarget = useRef(null);
  
  const loadMore = useCallback(async () => {
    if (loading || !hasMore) return; // Prevent duplicate loads
    
    setLoading(true);
    try {
      const newItems = await fetchItems(page);
      
      if (newItems.length === 0) {
        setHasMore(false);
      } else {
        setItems(prev => [...prev, ...newItems]);
        setPage(prev => prev + 1);
      }
    } catch (error) {
      console.error('Failed to load more:', error);
    } finally {
      setLoading(false);
    }
  }, [page, loading, hasMore]);
  
  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting) {
          loadMore();
        }
      },
      { threshold: 1.0 }
    );
    
    if (observerTarget.current) {
      observer.observe(observerTarget.current);
    }
    
    return () => observer.disconnect();
  }, [loadMore]);
  
  return (
    <div>
      {items.map(item => <Item key={item.id} item={item} />)}
      {hasMore && <div ref={observerTarget}>Loading...</div>}
    </div>
  );
}
```

**What this gets right:**
- ✅ Prevents duplicate loads with loading flag
- ✅ Stops when no more data
- ✅ Cleans up observer
- ✅ Uses IntersectionObserver (performant)

**What it still needs:**
- Error handling UI
- Retry mechanism
- Scroll position restoration
- Initial load state

## The Common Bugs

### Bug 1: Double/Triple Loading

**The Problem:**
```javascript
// ❌ This loads the same page multiple times
useEffect(() => {
  const observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting) {
      loadMore(); // Fires multiple times before loading completes!
    }
  });
  
  observer.observe(target);
}, [loadMore]); // loadMore changes every render = re-create observer
```

**What happens:**
1. User scrolls to bottom
2. IntersectionObserver fires → loadMore() starts
3. Component re-renders (state change)
4. Observer re-created with new loadMore reference
5. Still intersecting → fires again
6. Same page loads 2-3 times

**The Fix:**
```javascript
// ✅ Guard with loading flag
const loadMore = useCallback(async () => {
  if (loading || !hasMore) return; // CRITICAL
  setLoading(true);
  // ... load data
}, [loading, hasMore]);

// ✅ Or use a ref to track in-flight request
const loadingRef = useRef(false);

const loadMore = useCallback(async () => {
  if (loadingRef.current || !hasMore) return;
  loadingRef.current = true;
  setLoading(true);
  
  try {
    // ... load data
  } finally {
    loadingRef.current = false;
    setLoading(false);
  }
}, [hasMore]);
```

### Bug 2: Race Conditions with Fast Scrolling

**The Problem:**
```javascript
// ❌ User scrolls fast, loads pages out of order
async function loadMore() {
  setLoading(true);
  const data = await fetchItems(page); // What if page changed?
  setItems(prev => [...prev, ...data]); // Might be wrong page!
  setPage(prev => prev + 1);
  setLoading(false);
}
```

**Scenario:**
1. Page 1 loads → Start loading page 2 (slow network)
2. User keeps scrolling → Start loading page 3 (fast network)
3. Page 3 arrives first → Added to list
4. Page 2 arrives second → Added after page 3
5. Result: Items in wrong order

**The Fix:**
```javascript
// ✅ Cancel stale requests
function InfiniteList() {
  const [items, setItems] = useState([]);
  const [nextCursor, setNextCursor] = useState(null);
  const abortControllerRef = useRef(null);
  
  const loadMore = useCallback(async () => {
    // Cancel previous request
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }
    
    const controller = new AbortController();
    abortControllerRef.current = controller;
    
    try {
      const { items: newItems, nextCursor: cursor } = await fetchItems({
        cursor: nextCursor,
        signal: controller.signal
      });
      
      setItems(prev => [...prev, ...newItems]);
      setNextCursor(cursor);
    } catch (error) {
      if (error.name !== 'AbortError') {
        console.error(error);
      }
    }
  }, [nextCursor]);
  
  // Cleanup on unmount
  useEffect(() => {
    return () => {
      if (abortControllerRef.current) {
        abortControllerRef.current.abort();
      }
    };
  }, []);
}
```

### Bug 3: Scroll Position Lost on Navigation

**The Problem:**
```javascript
// User scrolls down, clicks an item, hits back button
// Scroll position resets to top, loses their place
```

**The Fix: Save & Restore Scroll Position**
```javascript
function InfiniteList() {
  const listRef = useRef(null);
  const location = useLocation();
  
  // Save scroll position before navigation
  useEffect(() => {
    const saveScrollPosition = () => {
      if (listRef.current) {
        sessionStorage.setItem(
          `scroll-${location.pathname}`,
          listRef.current.scrollTop
        );
      }
    };
    
    window.addEventListener('beforeunload', saveScrollPosition);
    return () => window.removeEventListener('beforeunload', saveScrollPosition);
  }, [location.pathname]);
  
  // Restore scroll position on mount
  useEffect(() => {
    const savedPosition = sessionStorage.getItem(`scroll-${location.pathname}`);
    if (savedPosition && listRef.current) {
      listRef.current.scrollTop = parseInt(savedPosition);
    }
  }, [location.pathname]);
  
  return <div ref={listRef}>{/* items */}</div>;
}
```

## The Performance Patterns

### Pattern 1: Virtualization for Long Lists

**The Problem:** 10,000 items in the DOM = browser dies

**The Solution: React Window**
```javascript
import { FixedSizeList } from 'react-window';
import InfiniteLoader from 'react-window-infinite-loader';

function VirtualizedInfiniteList() {
  const [items, setItems] = useState([]);
  const [hasMore, setHasMore] = useState(true);
  
  const loadMoreItems = async (startIndex, stopIndex) => {
    const newItems = await fetchItems(startIndex, stopIndex);
    setItems(prev => [...prev, ...newItems]);
    if (newItems.length === 0) setHasMore(false);
  };
  
  const isItemLoaded = index => !hasMore || index < items.length;
  
  return (
    <InfiniteLoader
      isItemLoaded={isItemLoaded}
      itemCount={hasMore ? items.length + 1 : items.length}
      loadMoreItems={loadMoreItems}
    >
      {({ onItemsRendered, ref }) => (
        <FixedSizeList
          height={600}
          itemCount={items.length}
          itemSize={100}
          onItemsRendered={onItemsRendered}
          ref={ref}
        >
          {({ index, style }) => (
            <div style={style}>
              {items[index] ? <Item item={items[index]} /> : 'Loading...'}
            </div>
          )}
        </FixedSizeList>
      )}
    </InfiniteLoader>
  );
}
```

**When to use:**
- Lists with 1000+ items
- Items with consistent heights
- Performance is critical

### Pattern 2: Intersection Observer Threshold

**Trigger loading before user hits bottom:**
```javascript
const observer = new IntersectionObserver(
  (entries) => {
    if (entries[0].isIntersecting) {
      loadMore();
    }
  },
  { 
    rootMargin: '200px', // Start loading 200px before bottom
    threshold: 0.1 
  }
);
```

**Why:** Users don't see loading spinner. Feels instant.

### Pattern 3: Debounce Scroll Events (Old Approach)

**Don't do this anymore:**
```javascript
// ❌ Old way: scroll event listener
window.addEventListener('scroll', () => {
  if (window.innerHeight + window.scrollY >= document.body.offsetHeight) {
    loadMore();
  }
});

// Problems:
// - Fires constantly while scrolling
// - Needs debouncing
// - Poor performance
// - Hard to clean up
```

**Use IntersectionObserver instead.** It's built for this.

## The Pagination Strategies

### Strategy 1: Offset-based (Simple, but problematic)

```javascript
// Offset = page * pageSize
const fetchItems = async (page, pageSize = 20) => {
  const offset = page * pageSize;
  return fetch(`/api/items?offset=${offset}&limit=${pageSize}`);
};
```

**Pros:** Simple, works with SQL `OFFSET/LIMIT`

**Cons:**
- **Duplicate items** if data changes (someone adds item before your offset)
- **Skipped items** if someone deletes item
- **Slow** on large offsets (database has to count through all rows)

### Strategy 2: Cursor-based (Production-ready)

```javascript
// Cursor = ID of last item
const fetchItems = async (cursor = null, pageSize = 20) => {
  const url = cursor 
    ? `/api/items?cursor=${cursor}&limit=${pageSize}`
    : `/api/items?limit=${pageSize}`;
  return fetch(url);
};

// Server returns:
{
  items: [...],
  nextCursor: 'item-id-here', // Last item ID
  hasMore: true
}

// Usage:
function InfiniteList() {
  const [items, setItems] = useState([]);
  const [nextCursor, setNextCursor] = useState(null);
  
  const loadMore = async () => {
    const { items: newItems, nextCursor, hasMore } = await fetchItems(nextCursor);
    setItems(prev => [...prev, ...newItems]);
    setNextCursor(nextCursor);
    setHasMore(hasMore);
  };
}
```

**Pros:**
- No duplicates/skips even if data changes
- Fast (database uses indexed cursor)
- Scales to millions of rows

**Cons:**
- Can't jump to arbitrary page
- Server must support it

**Use cursor-based for production.** Offset-based is for prototypes only.

## The React Query Version (Recommended)

**Stop managing pagination state yourself:**

```javascript
import { useInfiniteQuery } from '@tanstack/react-query';
import { useInView } from 'react-intersection-observer';

function InfiniteList() {
  const { ref, inView } = useInView();
  
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    isError,
    error
  } = useInfiniteQuery({
    queryKey: ['items'],
    queryFn: ({ pageParam = null }) => fetchItems(pageParam),
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
  });
  
  // Auto-load when in view
  useEffect(() => {
    if (inView && hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [inView, hasNextPage, isFetchingNextPage, fetchNextPage]);
  
  if (isLoading) return <Spinner />;
  if (isError) return <Error error={error} />;
  
  const items = data.pages.flatMap(page => page.items);
  
  return (
    <div>
      {items.map(item => <Item key={item.id} item={item} />)}
      
      {hasNextPage && (
        <div ref={ref}>
          {isFetchingNextPage ? 'Loading more...' : 'Load more'}
        </div>
      )}
    </div>
  );
}
```

**What React Query handles for you:**
- Caching (scroll away and back = instant)
- Deduplication (multiple components = one request)
- Background refetching
- Error retry
- Stale data management
- Request cancellation

**You just provide:** fetch function and cursor extraction.

## The Error Handling

**Don't just console.error and move on:**

```javascript
function InfiniteList() {
  const [error, setError] = useState(null);
  const [retrying, setRetrying] = useState(false);
  
  const loadMore = async () => {
    setError(null); // Clear previous error
    setLoading(true);
    
    try {
      const data = await fetchItems(nextCursor);
      // ... handle success
    } catch (err) {
      setError(err);
      // Don't clear loading state - allow retry
    } finally {
      setLoading(false);
    }
  };
  
  const retry = () => {
    setRetrying(true);
    loadMore().finally(() => setRetrying(false));
  };
  
  return (
    <div>
      {items.map(item => <Item key={item.id} item={item} />)}
      
      {error && (
        <div className="error">
          <p>Failed to load more items</p>
          <button onClick={retry} disabled={retrying}>
            {retrying ? 'Retrying...' : 'Retry'}
          </button>
        </div>
      )}
      
      {!error && hasMore && (
        <div ref={observerTarget}>
          {loading ? 'Loading...' : 'Load more'}
        </div>
      )}
    </div>
  );
}
```

## The UX Polish

### Show skeleton loaders, not spinners:
```javascript
{loading ? (
  Array(5).fill(null).map((_, i) => <ItemSkeleton key={i} />)
) : (
  items.map(item => <Item key={item.id} item={item} />)
)}
```

### Add "Back to top" button for long lists:
```javascript
const [showBackToTop, setShowBackToTop] = useState(false);

useEffect(() => {
  const handleScroll = () => {
    setShowBackToTop(window.scrollY > 1000);
  };
  
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);

{showBackToTop && (
  <button 
    className="back-to-top"
    onClick={() => window.scrollTo({ top: 0, behavior: 'smooth' })}
  >
    ↑ Back to top
  </button>
)}
```

### Show item count:
```javascript
<div className="list-header">
  Showing {items.length} items
  {hasMore && ' (loading more as you scroll)'}
</div>
```

## The Complete Example

Putting it all together:

```javascript
import { useInfiniteQuery } from '@tanstack/react-query';
import { useInView } from 'react-intersection-observer';

function ProductList() {
  const { ref, inView } = useInView({ threshold: 0.5 });
  
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    isError,
    error,
    refetch
  } = useInfiniteQuery({
    queryKey: ['products'],
    queryFn: async ({ pageParam = null }) => {
      const res = await fetch(
        `/api/products?cursor=${pageParam || ''}&limit=20`
      );
      if (!res.ok) throw new Error('Failed to fetch');
      return res.json();
    },
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    staleTime: 60000, // Cache for 1 minute
  });
  
  useEffect(() => {
    if (inView && hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [inView, hasNextPage, isFetchingNextPage, fetchNextPage]);
  
  if (isLoading) {
    return <div className="grid">{Array(12).fill(<ProductSkeleton />)}</div>;
  }
  
  if (isError) {
    return (
      <div className="error-state">
        <p>Failed to load products: {error.message}</p>
        <button onClick={() => refetch()}>Try Again</button>
      </div>
    );
  }
  
  const products = data.pages.flatMap(page => page.items);
  
  return (
    <div>
      <div className="list-header">
        {products.length} products
      </div>
      
      <div className="grid">
        {products.map(product => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
      
      {hasNextPage && (
        <div ref={ref} className="load-more">
          {isFetchingNextPage ? (
            <Spinner />
          ) : (
            <p>Scroll for more</p>
          )}
        </div>
      )}
      
      {!hasNextPage && products.length > 0 && (
        <div className="end-message">
          You've reached the end!
        </div>
      )}
    </div>
  );
}
```

## The Checklist

Before shipping infinite scroll:

- [ ] **Prevents duplicate loads** (loading flag or ref)
- [ ] **Handles race conditions** (AbortController)
- [ ] **Restores scroll position** on navigation
- [ ] **Uses cursor-based pagination** (not offset)
- [ ] **Shows loading state** (skeleton or spinner)
- [ ] **Handles errors gracefully** (retry button)
- [ ] **Indicates end of list** ("You've reached the end")
- [ ] **Uses IntersectionObserver** (not scroll events)
- [ ] **Cleans up observers** on unmount
- [ ] **Tests with slow 3G** (network throttling in DevTools)
- [ ] **Tests with fast scrolling** (scroll to bottom immediately)
- [ ] **Virtualizes if needed** (>1000 items)

## The Bottom Line

**Tutorial infinite scroll:** "Load more on scroll"  
**Production infinite scroll:** Race conditions, scroll restoration, error states, performance optimization, and graceful degradation

Use React Query + IntersectionObserver + cursor pagination. Everything else is reinventing wheels that are already round.

---

### Further Reading
- [Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) - MDN
- [React Query Infinite Queries](https://tanstack.com/query/latest/docs/framework/react/guides/infinite-queries) - Official docs
- [react-window](https://github.com/bvaughn/react-window) - Virtualization library
- [Cursor-based Pagination](https://www.apollographql.com/docs/react/pagination/cursor-based/) - Apollo GraphQL