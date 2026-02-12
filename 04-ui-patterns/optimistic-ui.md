# Optimistic UI: When Speed Beats Correctness (Sometimes)

**The principle:** Update the UI immediately, sync with the server later.

**The promise:** Instant feedback, snappy UX, users love it.

**The reality:** Rollbacks, race conditions, partial failures, and users who click faster than your server can respond.

## What Optimistic UI Actually Means

```javascript
// ❌ Pessimistic UI (wait for server)
async function handleLike() {
  setLoading(true);
  await api.likePost(postId);  // Wait...
  setLiked(true);               // Then update
  setLoading(false);
}
// User clicks → Spinner → 500ms → Heart fills
// Feels sluggish

// ✅ Optimistic UI (update immediately)
async function handleLike() {
  setLiked(true);  // Update NOW
  
  try {
    await api.likePost(postId);  // Sync in background
  } catch (error) {
    setLiked(false);  // Rollback if it fails
    showError('Failed to like post');
  }
}
// User clicks → Heart fills instantly → (server confirms in background)
// Feels instant
```

**The tradeoff:** Speed vs. truthfulness. You're showing a future you hope will happen.

## When to Use Optimistic UI

### ✅ Good Use Cases

**1. Low-stakes actions that almost never fail:**
- Liking/unliking posts
- Following/unfollowing users
- Marking notifications as read
- Simple toggles (dark mode, settings)
- Adding items to cart

**Why:** Failure rate < 1%, impact is low, rollback is simple.

**2. Actions with instant local feedback:**
- Typing in collaborative editors (like Google Docs)
- Moving items in a drag-and-drop list
- Checking off todos
- Deleting draft content

**Why:** User expects immediate visual response. Network delay breaks the illusion.

### ❌ Bad Use Cases

**1. Financial transactions:**
```javascript
// ❌ NEVER do this
function handlePayment() {
  setBalance(balance - amount);  // Show money gone
  await processPayment(amount);   // What if this fails?
}
```

**Why:** You just told the user they paid. If it fails, they'll (rightfully) be furious.

**2. Irreversible destructive actions:**
```javascript
// ❌ Don't do this
function handleDeleteAccount() {
  setAccountDeleted(true);  // Show account deleted
  await deleteAccount();     // What if network fails?
}
```

**Why:** User thinks their account is gone. Server still has it. Now what?

**3. Server-validated actions:**
```javascript
// ❌ Risky
function handleSubmitForm() {
  setSubmitted(true);  // Show success
  await submitForm();   // Server rejects: "Email already exists"
}
```

**Why:** You don't know if the server will accept it. High failure rate = bad UX.

**The rule:** If failure would be confusing, embarrassing, or costly to the user, don't use optimistic UI.

## The Basic Pattern

```javascript
function LikeButton({ postId, initialLiked }) {
  const [liked, setLiked] = useState(initialLiked);
  const [optimisticLike, setOptimisticLike] = useState(null);
  
  // Show optimistic state if it exists, otherwise actual state
  const displayLiked = optimisticLike ?? liked;
  
  const handleLike = async () => {
    const newState = !displayLiked;
    setOptimisticLike(newState);  // Optimistic update
    
    try {
      await api.likePost(postId, newState);
      setLiked(newState);  // Confirm with server response
      setOptimisticLike(null);  // Clear optimistic state
    } catch (error) {
      setOptimisticLike(null);  // Rollback
      showError('Failed to like post');
    }
  };
  
  return (
    <button 
      onClick={handleLike}
      className={displayLiked ? 'liked' : ''}
    >
      {displayLiked ? '❤️' : '🤍'}
    </button>
  );
}
```

**Key points:**
- Separate optimistic state from confirmed state
- Show optimistic state while request is in flight
- Clear optimistic state on success or failure

## The React Query Way (Recommended)

**Stop managing optimistic state manually:**

```javascript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function LikeButton({ postId, initialLiked }) {
  const queryClient = useQueryClient();
  
  const likeMutation = useMutation({
    mutationFn: (liked) => api.likePost(postId, liked),
    
    // Optimistic update
    onMutate: async (newLiked) => {
      // Cancel outgoing queries (avoid overwriting our optimistic update)
      await queryClient.cancelQueries(['post', postId]);
      
      // Snapshot current value
      const previousPost = queryClient.getQueryData(['post', postId]);
      
      // Optimistically update
      queryClient.setQueryData(['post', postId], (old) => ({
        ...old,
        liked: newLiked,
        likeCount: newLiked ? old.likeCount + 1 : old.likeCount - 1
      }));
      
      // Return context with snapshot
      return { previousPost };
    },
    
    // Rollback on error
    onError: (err, newLiked, context) => {
      queryClient.setQueryData(['post', postId], context.previousPost);
      showError('Failed to like post');
    },
    
    // Refetch on success (get real data from server)
    onSettled: () => {
      queryClient.invalidateQueries(['post', postId]);
    }
  });
  
  const { data: post } = useQuery(['post', postId]);
  
  return (
    <button onClick={() => likeMutation.mutate(!post.liked)}>
      {post.liked ? '❤️' : '🤍'} {post.likeCount}
    </button>
  );
}
```

**What React Query handles:**
- Cancels conflicting queries
- Snapshots state before optimistic update
- Automatically rolls back on error
- Refetches to confirm server state
- Handles race conditions

## The Real-World Patterns

### Pattern 1: List Item Deletion

```javascript
function TodoList() {
  const queryClient = useQueryClient();
  
  const deleteMutation = useMutation({
    mutationFn: (todoId) => api.deleteTodo(todoId),
    
    onMutate: async (todoId) => {
      await queryClient.cancelQueries(['todos']);
      
      const previousTodos = queryClient.getQueryData(['todos']);
      
      // Optimistically remove from list
      queryClient.setQueryData(['todos'], (old) =>
        old.filter(todo => todo.id !== todoId)
      );
      
      return { previousTodos };
    },
    
    onError: (err, todoId, context) => {
      // Rollback: restore the deleted item
      queryClient.setQueryData(['todos'], context.previousTodos);
      showError('Failed to delete');
    },
    
    onSettled: () => {
      queryClient.invalidateQueries(['todos']);
    }
  });
  
  const { data: todos } = useQuery(['todos'], fetchTodos);
  
  return (
    <div>
      {todos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onDelete={() => deleteMutation.mutate(todo.id)}
        />
      ))}
    </div>
  );
}
```

**UX Enhancement: Fade out before removing**
```javascript
function TodoItem({ todo, onDelete }) {
  const [deleting, setDeleting] = useState(false);
  
  const handleDelete = async () => {
    setDeleting(true);  // Start fade animation
    setTimeout(() => {
      onDelete();  // Remove from list after animation
    }, 300);
  };
  
  return (
    <div className={deleting ? 'fade-out' : ''}>
      {todo.text}
      <button onClick={handleDelete}>Delete</button>
    </div>
  );
}
```

### Pattern 2: Adding to List

```javascript
const addTodoMutation = useMutation({
  mutationFn: (text) => api.createTodo(text),
  
  onMutate: async (text) => {
    await queryClient.cancelQueries(['todos']);
    
    const previousTodos = queryClient.getQueryData(['todos']);
    
    // Optimistically add with temporary ID
    const optimisticTodo = {
      id: `temp-${Date.now()}`,  // Temporary ID
      text,
      completed: false,
      createdAt: new Date(),
      optimistic: true  // Flag for UI styling
    };
    
    queryClient.setQueryData(['todos'], (old) => [optimisticTodo, ...old]);
    
    return { previousTodos, optimisticId: optimisticTodo.id };
  },
  
  onSuccess: (serverTodo, text, context) => {
    // Replace optimistic todo with server version
    queryClient.setQueryData(['todos'], (old) =>
      old.map(todo => 
        todo.id === context.optimisticId ? serverTodo : todo
      )
    );
  },
  
  onError: (err, text, context) => {
    queryClient.setQueryData(['todos'], context.previousTodos);
    showError('Failed to add todo');
  }
});

// In component:
function TodoItem({ todo }) {
  return (
    <div className={todo.optimistic ? 'todo-pending' : ''}>
      {todo.text}
      {todo.optimistic && <Spinner size="small" />}
    </div>
  );
}
```

### Pattern 3: Inline Editing

```javascript
function EditableTitle({ postId, initialTitle }) {
  const [title, setTitle] = useState(initialTitle);
  const [editing, setEditing] = useState(false);
  const queryClient = useQueryClient();
  
  const updateMutation = useMutation({
    mutationFn: (newTitle) => api.updatePost(postId, { title: newTitle }),
    
    onMutate: async (newTitle) => {
      await queryClient.cancelQueries(['post', postId]);
      
      const previousPost = queryClient.getQueryData(['post', postId]);
      
      queryClient.setQueryData(['post', postId], (old) => ({
        ...old,
        title: newTitle
      }));
      
      return { previousPost, previousTitle: title };
    },
    
    onError: (err, newTitle, context) => {
      queryClient.setQueryData(['post', postId], context.previousPost);
      setTitle(context.previousTitle);  // Restore local state
      showError('Failed to update title');
    },
    
    onSettled: () => {
      setEditing(false);
    }
  });
  
  const handleSave = () => {
    if (title !== initialTitle) {
      updateMutation.mutate(title);
    } else {
      setEditing(false);
    }
  };
  
  return editing ? (
    <input
      value={title}
      onChange={(e) => setTitle(e.target.value)}
      onBlur={handleSave}
      onKeyDown={(e) => e.key === 'Enter' && handleSave()}
      autoFocus
    />
  ) : (
    <h1 onClick={() => setEditing(true)}>
      {title}
    </h1>
  );
}
```

## The Edge Cases

### Edge Case 1: Multiple Rapid Clicks

```javascript
// ❌ Problem: User clicks like button 5 times in 1 second
function LikeButton() {
  const [liked, setLiked] = useState(false);
  
  const handleLike = async () => {
    const newState = !liked;
    setLiked(newState);
    await api.likePost(postId, newState);
  };
  
  // Clicks: unlike, like, unlike, like, unlike
  // Server gets: 5 conflicting requests
  // Final state: Unpredictable
}

// ✅ Solution: Debounce or disable during mutation
function LikeButton() {
  const likeMutation = useMutation({
    mutationFn: (liked) => api.likePost(postId, liked)
  });
  
  return (
    <button 
      onClick={() => likeMutation.mutate(!liked)}
      disabled={likeMutation.isLoading}  // Prevent spam
    >
      {liked ? '❤️' : '🤍'}
    </button>
  );
}

// Or debounce:
const debouncedLike = useMemo(
  () => debounce((liked) => likeMutation.mutate(liked), 300),
  []
);
```

### Edge Case 2: Partial Failures

```javascript
// User adds comment with image upload
// Image upload succeeds, comment creation fails
// What do you rollback?

const addCommentMutation = useMutation({
  mutationFn: async (data) => {
    // Step 1: Upload image
    const imageUrl = await uploadImage(data.image);
    
    // Step 2: Create comment
    return api.createComment({ text: data.text, imageUrl });
  },
  
  onMutate: (data) => {
    // Optimistically show comment with uploading image
    const optimisticComment = {
      id: `temp-${Date.now()}`,
      text: data.text,
      imageUrl: URL.createObjectURL(data.image),  // Local preview
      uploading: true
    };
    
    queryClient.setQueryData(['comments'], (old) => 
      [optimisticComment, ...old]
    );
    
    return { optimisticId: optimisticComment.id };
  },
  
  onSuccess: (serverComment, data, context) => {
    // Replace optimistic comment with server version
    queryClient.setQueryData(['comments'], (old) =>
      old.map(c => c.id === context.optimisticId ? serverComment : c)
    );
  },
  
  onError: (err, data, context) => {
    // Remove failed comment
    queryClient.setQueryData(['comments'], (old) =>
      old.filter(c => c.id !== context.optimisticId)
    );
    
    // Image already uploaded to server but comment failed
    // Options:
    // 1. Delete the uploaded image (complex)
    // 2. Accept orphaned image (simple, but wasteful)
    // 3. Don't optimistically update images (safest)
    
    showError('Failed to post comment');
  }
});
```

**Lesson:** Optimistic UI gets complex with multi-step operations. Sometimes pessimistic is better.

### Edge Case 3: Concurrent Edits

```javascript
// User A and User B both edit the same document
// User A: Optimistically updates to "Hello World"
// User B: Optimistically updates to "Goodbye World"
// Server: Last write wins
// Result: One user's changes disappear

// Solution: Operational Transform or CRDT (beyond scope)
// Or: Don't use optimistic UI for collaborative editing
// Or: Lock editing (only one editor at a time)
```

## The Rollback Strategies

### Strategy 1: Full Rollback (Simplest)
```javascript
onError: (err, variables, context) => {
  // Restore entire previous state
  queryClient.setQueryData(queryKey, context.previousData);
}
```

**When:** Simple toggles, single-field updates

### Strategy 2: Partial Rollback
```javascript
onError: (err, variables, context) => {
  // Only rollback the failed part
  queryClient.setQueryData(queryKey, (current) => ({
    ...current,
    failedField: context.previousData.failedField
  }));
}
```

**When:** Multi-field forms, complex state

### Strategy 3: No Rollback (Show Error State)
```javascript
onMutate: (newData) => {
  // Mark as pending
  queryClient.setQueryData(queryKey, (old) => ({
    ...old,
    ...newData,
    status: 'pending'
  }));
},

onError: (err, newData, context) => {
  // Mark as failed, but keep the attempted changes
  queryClient.setQueryData(queryKey, (old) => ({
    ...old,
    status: 'error',
    error: err.message
  }));
}
```

**When:** User needs to see what failed to fix it (form validation errors)

## The Error UX

**Don't just rollback silently:**

```javascript
// ❌ User has no idea what happened
onError: (err, variables, context) => {
  queryClient.setQueryData(queryKey, context.previousData);
}

// ✅ Show what failed and why
onError: (err, variables, context) => {
  queryClient.setQueryData(queryKey, context.previousData);
  
  toast.error(
    <div>
      <strong>Failed to save changes</strong>
      <p>{err.message}</p>
      <button onClick={() => retry()}>Try Again</button>
    </div>
  );
}
```

**Or keep the optimistic state visible as "pending save":**

```javascript
function SaveIndicator({ status, error }) {
  if (status === 'saving') return <span>💾 Saving...</span>;
  if (status === 'error') return <span>⚠️ Not saved. {error}</span>;
  if (status === 'saved') return <span>✓ Saved</span>;
  return null;
}
```

## The Testing Strategy

**Optimistic UI is hard to test. Here's how:**

```javascript
// Test 1: Happy path
test('optimistically updates like count', async () => {
  render(<LikeButton postId="1" />);
  
  const button = screen.getByRole('button');
  await userEvent.click(button);
  
  // Should show liked immediately
  expect(button).toHaveClass('liked');
  
  // Wait for server confirmation
  await waitFor(() => {
    expect(mockApi.likePost).toHaveBeenCalled();
  });
});

// Test 2: Error handling
test('rolls back on error', async () => {
  mockApi.likePost.mockRejectedValue(new Error('Network error'));
  
  render(<LikeButton postId="1" />);
  
  const button = screen.getByRole('button');
  await userEvent.click(button);
  
  // Should show liked optimistically
  expect(button).toHaveClass('liked');
  
  // Should rollback after error
  await waitFor(() => {
    expect(button).not.toHaveClass('liked');
  });
  
  // Should show error
  expect(screen.getByText(/failed/i)).toBeInTheDocument();
});

// Test 3: Rapid clicks
test('handles rapid clicks', async () => {
  render(<LikeButton postId="1" />);
  
  const button = screen.getByRole('button');
  
  // Click 5 times rapidly
  await userEvent.click(button);
  await userEvent.click(button);
  await userEvent.click(button);
  await userEvent.click(button);
  await userEvent.click(button);
  
  // Should only send one request (or handle all correctly)
  await waitFor(() => {
    expect(mockApi.likePost).toHaveBeenCalledTimes(1);
  });
});
```

## The Checklist

Before shipping optimistic UI:

- [ ] **Low failure rate** (<1% expected)
- [ ] **Low stakes** (failure isn't catastrophic)
- [ ] **Easy rollback** (can undo the optimistic change)
- [ ] **Error UI** (user knows if it failed)
- [ ] **Retry mechanism** (user can try again)
- [ ] **Prevents double-clicks** (disable or debounce)
- [ ] **Handles race conditions** (uses React Query or similar)
- [ ] **Tests rollback** (simulate network errors)
- [ ] **Tests rapid actions** (spam clicks)
- [ ] **Indicates pending state** (if background sync is slow)

## The Bottom Line

**Optimistic UI is not a silver bullet.**

Use it for:
- High-frequency, low-stakes interactions
- Actions that users expect to be instant
- Read-dominated features (likes, follows)

Don't use it for:
- Financial transactions
- Destructive actions
- Server-validated operations
- Collaborative editing (without CRDT/OT)

When in doubt, pessimistic UI is fine. A 200ms delay is better than a confusing rollback.

**The best optimistic UI is the one the user never notices—because it never fails.**

---

### Further Reading
- [Optimistic Updates with React Query](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates) - Official docs
- [Optimistic UI Patterns](https://www.apollographql.com/docs/react/performance/optimistic-ui/) - Apollo GraphQL
- [CRDTs for Conflict-Free Editing](https://crdt.tech/) - For collaborative features
- [The UX of Optimistic UI](https://www.smashingmagazine.com/2016/11/true-lies-of-optimistic-user-interfaces/) - Smashing Magazine