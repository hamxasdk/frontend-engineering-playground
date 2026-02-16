# Notification Flow: End-to-End System Design

**The challenge:** User performs action → All relevant users get notified across all their devices, immediately.

**The reality:** This is harder than it sounds. Let me show you how to build it right.

## The Complete Flow (30,000 Foot View)

```
Event Occurs → Create Notification → Route to Users → Deliver to Devices → Update UI
     ↓               ↓                    ↓                  ↓              ↓
(User posts)    (Save to DB)      (Find followers)    (Push, Email)   (Badge, Toast)
```

Simple in theory. Complex in practice.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Client Layer                                  │
│                                                                          │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐  │
│  │   In-App   │  │   Browser    │  │   Mobile    │  │    Email     │  │
│  │   Toast    │  │   Push API   │  │   Push (FCM)│  │   Worker     │  │
│  └────────────┘  └──────────────┘  └─────────────┘  └──────────────┘  │
│        ↑                ↑                  ↑                 ↑          │
└────────┼────────────────┼──────────────────┼─────────────────┼──────────┘
         │                │                  │                 │
         └────────────────┴──────────────────┴─────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                      Notification Service Layer                         │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Notification Router                                              │  │
│  │  - User preferences (push, email, in-app)                        │  │
│  │  - Device management (active devices per user)                   │  │
│  │  - Priority routing (urgent vs normal)                           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Delivery Queue (Redis/RabbitMQ)                                 │  │
│  │  - Batching & rate limiting                                      │  │
│  │  - Priority queues (urgent, normal, low)                         │  │
│  │  - Retry logic with exponential backoff                          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Notification Store (PostgreSQL + Redis)                         │  │
│  │  - Persistent storage: PostgreSQL                                │  │
│  │  - Unread counts: Redis                                          │  │
│  │  - Real-time updates: Redis Pub/Sub                              │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↑
┌─────────────────────────────────────────────────────────────────────────┐
│                         Event Source Layer                              │
│                                                                          │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────────┐        │
│  │   User    │  │  Comment  │  │   Like    │  │   System     │        │
│  │  Actions  │  │  Service  │  │  Service  │  │   Events     │        │
│  └───────────┘  └───────────┘  └───────────┘  └──────────────┘        │
└─────────────────────────────────────────────────────────────────────────┘
```

## Pattern 1: Event-Driven Notification Creation

**Don't create notifications inline. Use events.**

### ❌ Bad: Inline Notification Creation
```javascript
async function createPost(userId, content) {
  const post = await db.posts.create({ userId, content });
  
  // BAD: Blocking the response
  const followers = await db.followers.find({ followingId: userId });
  
  for (const follower of followers) {
    await db.notifications.create({
      userId: follower.userId,
      type: 'new_post',
      postId: post.id
    });
  }
  
  return post; // User waits for all notifications to be created
}
```

**Problem:** If user has 10,000 followers, this takes seconds. HTTP request times out.

### ✅ Good: Event-Driven
```javascript
async function createPost(userId, content) {
  const post = await db.posts.create({ userId, content });
  
  // Emit event (non-blocking)
  await eventBus.publish('post.created', {
    postId: post.id,
    authorId: userId,
    timestamp: Date.now()
  });
  
  return post; // Instant response
}

// Separate worker handles notifications
eventBus.subscribe('post.created', async (event) => {
  await notificationService.createForEvent('new_post', event);
});
```

**Result:** API responds instantly. Notifications created in background.

## Pattern 2: Fan-Out Strategies

**Problem:** User with 1 million followers posts → Need 1 million notifications.

### Strategy A: Fan-Out on Write (Eager)

```javascript
// When post created, immediately create all notifications
async function handlePostCreated(event) {
  const followers = await db.followers.find({ 
    followingId: event.authorId 
  });
  
  // Batch insert (more efficient than one-by-one)
  await db.notifications.batchInsert(
    followers.map(follower => ({
      userId: follower.userId,
      type: 'new_post',
      data: { postId: event.postId },
      createdAt: new Date()
    }))
  );
  
  // Update unread counts in Redis
  await redis.pipeline(
    followers.map(f => redis.incr(`unread:${f.userId}`))
  );
}
```

**Pros:** Reading notifications is fast (already created)  
**Cons:** Writing is slow, storage cost (1M rows per post)

**Use for:** Users with <10k followers

### Strategy B: Fan-Out on Read (Lazy)

```javascript
// Store reference to post, create notifications on demand
async function handlePostCreated(event) {
  // Just store a single "feed item"
  await db.feedItems.create({
    authorId: event.authorId,
    postId: event.postId,
    type: 'post',
    createdAt: new Date()
  });
  
  // Invalidate follower feeds (they'll rebuild on next read)
  const followers = await db.followers.find({ 
    followingId: event.authorId 
  });
  
  await redis.pipeline(
    followers.map(f => redis.del(`feed:${f.userId}`))
  );
}

// When user checks notifications, build feed on-the-fly
async function getNotifications(userId) {
  const cached = await redis.get(`feed:${userId}`);
  if (cached) return JSON.parse(cached);
  
  // Build feed from followed users' feed items
  const following = await db.followers.find({ userId });
  const feedItems = await db.feedItems.find({
    authorId: { $in: following.map(f => f.followingId) },
    createdAt: { $gt: thirtyDaysAgo }
  }).sort({ createdAt: -1 }).limit(50);
  
  await redis.setex(`feed:${userId}`, 300, JSON.stringify(feedItems));
  return feedItems;
}
```

**Pros:** Writing is instant, less storage  
**Cons:** Reading is slower (compute per read)

**Use for:** Celebrity accounts (1M+ followers)

### Strategy C: Hybrid (Smart)

```javascript
async function handlePostCreated(event) {
  const followerCount = await db.followers.count({ 
    followingId: event.authorId 
  });
  
  if (followerCount < 10000) {
    // Fan-out on write for small audiences
    await fanOutOnWrite(event);
  } else {
    // Fan-out on read for large audiences
    await fanOutOnRead(event);
  }
}
```

**Best of both worlds.** Use fan-out on write for most users, fan-out on read for celebrities.

## Pattern 3: Notification Routing & Delivery

**Not all notifications should be delivered the same way.**

### Notification Priority System

```javascript
const NOTIFICATION_CONFIG = {
  'direct_message': {
    priority: 'urgent',
    channels: ['push', 'email', 'in_app'],
    batchable: false,
    sound: true
  },
  'new_comment': {
    priority: 'normal',
    channels: ['push', 'in_app'],
    batchable: true,
    batchWindow: 300000 // 5 minutes
  },
  'new_follower': {
    priority: 'low',
    channels: ['in_app'],
    batchable: true,
    batchWindow: 3600000 // 1 hour
  },
  'marketing': {
    priority: 'low',
    channels: ['email'],
    batchable: true,
    respectQuietHours: true
  }
};

async function routeNotification(notification) {
  const config = NOTIFICATION_CONFIG[notification.type];
  const user = await getUserPreferences(notification.userId);
  
  // Respect user preferences
  const channels = config.channels.filter(c => user.channels[c]);
  
  // Check quiet hours
  if (config.respectQuietHours && isQuietHours(user)) {
    channels = channels.filter(c => c !== 'push');
  }
  
  // Batch or send immediately
  if (config.batchable) {
    await addToBatch(notification, config.batchWindow);
  } else {
    await deliverNow(notification, channels);
  }
}
```

### Batching Logic

```javascript
// Batch notifications: "John, Mary, and 5 others liked your post"
class NotificationBatcher {
  constructor() {
    this.batches = new Map(); // userId → type → notifications[]
    this.timers = new Map();
  }
  
  add(notification, batchWindow) {
    const key = `${notification.userId}:${notification.type}`;
    
    if (!this.batches.has(key)) {
      this.batches.set(key, []);
      
      // Flush after batch window
      this.timers.set(key, setTimeout(() => {
        this.flush(key);
      }, batchWindow));
    }
    
    this.batches.get(key).push(notification);
  }
  
  async flush(key) {
    const notifications = this.batches.get(key);
    this.batches.delete(key);
    clearTimeout(this.timers.get(key));
    this.timers.delete(key);
    
    if (notifications.length === 1) {
      // Send individual notification
      await deliver(notifications[0]);
    } else {
      // Send batched notification
      await deliver({
        type: notifications[0].type,
        userId: notifications[0].userId,
        count: notifications.length,
        actors: notifications.map(n => n.actorId).slice(0, 3),
        data: notifications[0].data
      });
    }
  }
}
```

**Result:** Instead of 20 separate "X liked your post" notifications, user gets one "John, Sarah and 18 others liked your post".

## Pattern 4: Multi-Device Delivery

**User has 3 devices. Notification should appear on all, but only notify once.**

### Device Registry

```javascript
// Schema
{
  userId: 'user-123',
  devices: [
    {
      deviceId: 'device-1',
      type: 'web',
      pushToken: 'fcm-token-...',
      lastActive: '2024-01-15T10:30:00Z',
      userAgent: 'Chrome/120...'
    },
    {
      deviceId: 'device-2',
      type: 'ios',
      pushToken: 'apns-token-...',
      lastActive: '2024-01-15T09:00:00Z'
    }
  ]
}
```

### Smart Delivery Logic

```javascript
async function deliverToDevices(notification, userId) {
  const devices = await getActiveDevices(userId);
  
  // Determine "primary" device (most recently active)
  const primary = devices.sort((a, b) => 
    b.lastActive - a.lastActive
  )[0];
  
  // Deliver to all devices, but only notify on primary
  const deliveries = devices.map(device => ({
    device,
    notification: {
      ...notification,
      // Only the primary device shows alert/sound/badge
      silent: device.deviceId !== primary.deviceId
    }
  }));
  
  await Promise.all(
    deliveries.map(d => pushToDevice(d.device, d.notification))
  );
}
```

**Result:** Notification appears in notification center on all devices, but only makes noise/vibration on the device you're actively using.

## Pattern 5: Real-Time Sync with WebSocket

**User marks notification as read on phone → Should update on desktop instantly.**

### Client-Side State Management

```javascript
function useNotifications() {
  const [notifications, setNotifications] = useState([]);
  const [unreadCount, setUnreadCount] = useState(0);
  const ws = useWebSocket();
  
  // Fetch initial notifications
  const { data } = useQuery('notifications', fetchNotifications);
  
  useEffect(() => {
    if (data) {
      setNotifications(data.notifications);
      setUnreadCount(data.unreadCount);
    }
  }, [data]);
  
  // Listen for real-time updates
  useEffect(() => {
    ws.on('notification:new', (notif) => {
      setNotifications(prev => [notif, ...prev]);
      setUnreadCount(prev => prev + 1);
    });
    
    ws.on('notification:read', ({ notificationId }) => {
      setNotifications(prev =>
        prev.map(n => n.id === notificationId ? { ...n, read: true } : n)
      );
      setUnreadCount(prev => Math.max(0, prev - 1));
    });
    
    ws.on('notification:read_all', () => {
      setNotifications(prev => prev.map(n => ({ ...n, read: true })));
      setUnreadCount(0);
    });
    
    return () => {
      ws.off('notification:new');
      ws.off('notification:read');
      ws.off('notification:read_all');
    };
  }, [ws]);
  
  const markAsRead = async (notificationId) => {
    // Optimistic update
    setNotifications(prev =>
      prev.map(n => n.id === notificationId ? { ...n, read: true } : n)
    );
    setUnreadCount(prev => Math.max(0, prev - 1));
    
    // Send to server
    await api.markNotificationAsRead(notificationId);
  };
  
  return { notifications, unreadCount, markAsRead };
}
```

### Server-Side Broadcast

```javascript
async function markAsRead(userId, notificationId) {
  // Update database
  await db.notifications.update(
    { id: notificationId, userId },
    { read: true, readAt: new Date() }
  );
  
  // Update Redis cache
  await redis.decr(`unread:${userId}`);
  
  // Broadcast to all user's devices
  await redis.publish(`user:${userId}`, JSON.stringify({
    type: 'notification:read',
    notificationId
  }));
}

// All WebSocket servers listen
redis.subscribe('user:*');
redis.on('message', (channel, message) => {
  const userId = channel.split(':')[1];
  const data = JSON.parse(message);
  
  // Send to all connected sockets for this user
  io.to(`user:${userId}`).emit(data.type, data);
});
```

**Result:** Mark as read on phone → Desktop updates instantly without polling.

## Pattern 6: Permission Handling

**Push notifications require explicit user permission. Handle this gracefully.**

### Progressive Permission Request

```javascript
function useNotificationPermission() {
  const [permission, setPermission] = useState(Notification.permission);
  const [prompted, setPrompted] = useState(false);
  
  const requestPermission = async () => {
    if (permission === 'granted') return true;
    if (permission === 'denied') return false;
    
    setPrompted(true);
    const result = await Notification.requestPermission();
    setPermission(result);
    
    if (result === 'granted') {
      // Register for push notifications
      const subscription = await registerPushSubscription();
      await api.registerDevice(subscription);
    }
    
    return result === 'granted';
  };
  
  return { permission, requestPermission, prompted };
}

// Don't ask immediately on page load - wait for user engagement
function NotificationPrompt() {
  const { permission, requestPermission } = useNotificationPermission();
  const [dismissed, setDismissed] = useState(false);
  
  // Only show after user has been active for 2 minutes
  const [showPrompt, setShowPrompt] = useState(false);
  useEffect(() => {
    const timer = setTimeout(() => setShowPrompt(true), 120000);
    return () => clearTimeout(timer);
  }, []);
  
  if (permission === 'granted' || permission === 'denied' || dismissed || !showPrompt) {
    return null;
  }
  
  return (
    <Banner>
      Get notified when someone replies to you
      <button onClick={requestPermission}>Enable</button>
      <button onClick={() => setDismissed(true)}>Later</button>
    </Banner>
  );
}
```

**Key insight:** Don't ask for permission immediately. Wait until user is engaged and you can show value.

## Pattern 7: Rate Limiting & Anti-Spam

**Prevent notification spam that drives users to disable all notifications.**

### Server-Side Rate Limiting

```javascript
class NotificationRateLimiter {
  constructor() {
    this.redis = redis;
  }
  
  async canSend(userId, notificationType) {
    const rules = {
      'new_follower': { limit: 20, window: 3600 }, // 20/hour
      'new_comment': { limit: 50, window: 3600 },  // 50/hour
      'new_like': { limit: 100, window: 3600 },    // 100/hour
      'direct_message': { limit: 1000, window: 3600 } // No real limit
    };
    
    const rule = rules[notificationType];
    if (!rule) return true;
    
    const key = `ratelimit:${userId}:${notificationType}`;
    const count = await this.redis.incr(key);
    
    if (count === 1) {
      await this.redis.expire(key, rule.window);
    }
    
    return count <= rule.limit;
  }
  
  async shouldBatch(userId, notificationType) {
    const count = await this.redis.get(`ratelimit:${userId}:${notificationType}`);
    
    // If hitting rate limit, start batching
    return parseInt(count) > 10;
  }
}

async function createNotification(userId, type, data) {
  const limiter = new NotificationRateLimiter();
  
  if (!await limiter.canSend(userId, type)) {
    console.log('Rate limit hit, skipping notification');
    return;
  }
  
  if (await limiter.shouldBatch(userId, type)) {
    await batcher.add({ userId, type, data });
  } else {
    await deliver({ userId, type, data });
  }
}
```

### Client-Side Deduplication

```javascript
// Prevent showing same notification multiple times
function NotificationToast({ notification }) {
  const shownIds = useRef(new Set());
  const [toasts, setToasts] = useState([]);
  
  useEffect(() => {
    if (shownIds.current.has(notification.id)) {
      return; // Already shown
    }
    
    shownIds.current.add(notification.id);
    
    setToasts(prev => [...prev, notification]);
    
    // Auto-dismiss after 5 seconds
    setTimeout(() => {
      setToasts(prev => prev.filter(t => t.id !== notification.id));
    }, 5000);
  }, [notification]);
  
  return (
    <div>
      {toasts.map(toast => (
        <Toast key={toast.id} {...toast} />
      ))}
    </div>
  );
}
```

## Pattern 8: Notification Grouping

**Don't show 50 separate notifications. Group them intelligently.**

### Smart Grouping Algorithm

```javascript
async function groupNotifications(notifications) {
  const groups = {};
  
  for (const notif of notifications) {
    // Group key: type + related entity
    const key = `${notif.type}:${notif.data.postId || notif.data.commentId}`;
    
    if (!groups[key]) {
      groups[key] = {
        type: notif.type,
        actors: [],
        data: notif.data,
        latest: notif.createdAt,
        count: 0
      };
    }
    
    groups[key].actors.push(notif.actorId);
    groups[key].count++;
    groups[key].latest = Math.max(groups[key].latest, notif.createdAt);
  }
  
  return Object.values(groups).map(group => ({
    ...group,
    message: formatGroupMessage(group)
  }));
}

function formatGroupMessage(group) {
  const actors = group.actors.slice(0, 3);
  const remaining = group.count - actors.length;
  
  if (group.count === 1) {
    return `${actors[0]} ${ACTION_TEXT[group.type]}`;
  }
  
  if (group.count === 2) {
    return `${actors[0]} and ${actors[1]} ${ACTION_TEXT[group.type]}`;
  }
  
  if (remaining > 0) {
    return `${actors.join(', ')} and ${remaining} others ${ACTION_TEXT[group.type]}`;
  }
  
  return `${actors.join(', ')} ${ACTION_TEXT[group.type]}`;
}
```

**Result:**
- "John liked your post"
- "John and Mary liked your post"
- "John, Mary, and 5 others liked your post"

## Complete End-to-End Flow Example

**User A comments on User B's post:**

1. **Event Creation** (10ms)
   ```javascript
   await eventBus.publish('comment.created', {
     commentId: 'comment-123',
     postId: 'post-456',
     authorId: 'user-A',
     postAuthorId: 'user-B'
   });
   ```

2. **Notification Creation** (50ms)
   ```javascript
   const notification = await db.notifications.create({
     userId: 'user-B',
     type: 'new_comment',
     actorId: 'user-A',
     data: { commentId: 'comment-123', postId: 'post-456' },
     createdAt: new Date()
   });
   ```

3. **Routing Decision** (5ms)
   ```javascript
   const channels = await getDeliveryChannels('user-B', 'new_comment');
   // Result: ['push', 'in_app']
   ```

4. **Multi-Device Delivery** (100ms)
   ```javascript
   const devices = await getActiveDevices('user-B');
   // Devices: [iPhone, Desktop Chrome]
   await Promise.all([
     sendPushToDevice(devices[0], notification),
     sendPushToDevice(devices[1], notification)
   ]);
   ```

5. **Real-Time Update** (20ms)
   ```javascript
   await redis.publish('user:user-B', {
     type: 'notification:new',
     notification
   });
   // All WebSocket connections for User B receive update
   ```

6. **UI Update** (instant)
   ```javascript
   // Client receives WebSocket message
   setNotifications(prev => [notification, ...prev]);
   setUnreadCount(prev => prev + 1);
   // Badge updates, toast appears
   ```

**Total time:** ~185ms from event to user seeing notification.

## The Monitoring & Debugging Toolkit

### Key Metrics to Track

```javascript
// Delivery metrics
metrics.increment('notifications.created', { type: notification.type });
metrics.increment('notifications.delivered', { channel: 'push' });
metrics.increment('notifications.failed', { channel: 'push', reason: 'invalid_token' });

// Timing metrics
metrics.timing('notifications.delivery_time', deliveryTime);
metrics.timing('notifications.creation_to_read', readTime);

// Business metrics
metrics.increment('notifications.clicked', { type: notification.type });
metrics.increment('notifications.dismissed');
```

### Debug Logging

```javascript
logger.info('Notification created', {
  notificationId: notification.id,
  userId: notification.userId,
  type: notification.type,
  channels: channels,
  timestamp: Date.now()
});

logger.info('Notification delivered', {
  notificationId: notification.id,
  deviceId: device.id,
  channel: 'push',
  latency: Date.now() - notification.createdAt
});
```

## Common Pitfalls

### ❌ No Idempotency
```javascript
// BAD: Creating duplicate notifications
eventBus.subscribe('comment.created', async (event) => {
  await createNotification(event); // Can run multiple times!
});
```

**Fix:** Add idempotency key:
```javascript
const notificationId = hash(event.type + event.commentId + event.userId);
const exists = await db.notifications.exists({ id: notificationId });
if (!exists) {
  await createNotification({ id: notificationId, ...event });
}
```

### ❌ Blocking on Delivery
```javascript
// BAD: Waiting for push delivery
await sendPushNotification(device, notification); // Can take seconds!
return { success: true };
```

**Fix:** Queue it:
```javascript
await deliveryQueue.add({ device, notification });
return { success: true }; // Instant response
```

### ❌ No Cleanup
```javascript
// BAD: Infinite notification growth
await db.notifications.create(notification);
// Never deleted, table grows forever
```

**Fix:** Auto-cleanup:
```javascript
// Delete read notifications after 30 days
cron.schedule('0 0 * * *', async () => {
  await db.notifications.delete({
    read: true,
    createdAt: { $lt: thirtyDaysAgo }
  });
});
```

## The Bottom Line

**Notification systems are deceptively complex.**

The flow looks simple:
- Event happens → Create notification → Deliver it

But production requirements add layers:
- **Fan-out** (1 event → N notifications)
- **Routing** (which channels? which devices?)
- **Batching** (group related notifications)
- **Rate limiting** (don't spam users)
- **Multi-device sync** (real-time state across devices)
- **Permissions** (respect user choices)

**The secret:** Build in layers, make each layer async, monitor everything.

---

### Further Reading
- [Push Notification Best Practices](https://web.dev/push-notifications-overview/) - web.dev
- [FCM Architecture](https://firebase.google.com/docs/cloud-messaging/concept-options) - Firebase
- [Notification Fatigue](https://www.nngroup.com/articles/push-notification/) - Nielsen Norman Group
- [Building Notification Systems at Scale](https://www.uber.com/blog/notification-service/) - Uber Engineering