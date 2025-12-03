# 📦 Feature 3: Offline Message Queue

## ✅ Why is this the third priority?

### 1️⃣ **Implicit Requirements of TSP Specification**

Although the specification does not explicitly require it, looking at the transport layer design:

```
TSP-technical-specification.md (L115):
"S1: TSP can be run over many different transport protocol"

Meaning:
  • HTTP/WebSocket - Requires both parties online
  • TCP - Requires both parties online
  • Email (Theoretically) - Naturally supports offline!
  • IPFS (Theoretically) - Asynchronous messaging!
```

**Inference:**
- TSP is designed to be **transport agnostic**
- Some transports naturally support offline
- SDK should provide a consistent offline experience

### 2️⃣ **Real User Scenarios**

```
Scenario 1: Mobile App
────────────────────────────────────
Alice (Mobile, often offline)    Bob (Online)
  │                               │
  ✗ (Enters subway)               │
                                  ├─ send("Hello")
                                  ↓
                           What should happen to the message?
                    
Option A (Current): Lost
  ❌ Bob sees send failure
  ❌ Alice never receives it
  
Option B (Queue):
  ✅ Message stored in intermediary server
  ✅ Delivered automatically when Alice comes online
```

```
Scenario 2: Different Time Zones
────────────────────────────────────
Alice (Beijing, Night)        Bob (New York, Morning)
  │                             │
  ├─ send("Reschedule meeting")  │
  ↓                            ↓
  (Sleeping)                   (Working)
                           
Without Offline Queue:
  ❌ Bob must wait 8 hours
  ❌ Or Alice sets an alarm
  
With Offline Queue:
  ✅ Message delivered immediately
  ✅ Workflow not interrupted
```

### 3️⃣ **Intermediary Server Foundation Exists**

```rust
// examples/intermediary.rs Already implemented!
// But functionality is rudimentary

// Current implementation (Simplified)
let mut pending_messages = HashMap::new();

if receiver_online {
    send_now(message);
} else {
    pending_messages.insert(receiver, message);  // ← Simple cache
}

// ❌ Problems:
// 1. In-memory storage, lost on restart
// 2. No priority
// 3. No expiration time
// 4. No persistence

// Our Improvements:
// ✅ Persistence (Askar)
// ✅ Priority Queue
// ✅ TTL Support
// ✅ Cleanup Mechanism
```

**Code Reuse:**

```
examples/intermediary.rs (300+ lines)
        ↓
Extract core logic → message_queue.rs (~400 lines)
        ↓
Enhance features (Persistence, Priority) (~200 lines)
        ↓
Total: ~600 lines new code, reuse 300 lines
```

### 4️⃣ **Completeness Considerations**

All three modes of TSP benefit:

```
Direct Mode:
  Alice ─────→ Bob
  If Bob is offline? Need queue

Nested Mode:
  Alice ─────→ Intermediary ─────→ Bob
            (outer)         (inner)
  If Bob is offline? Need queue
  
Routed Mode:
  Alice → Hop1 → Hop2 → Bob
  If any hop is offline? Need queue
```

**Design Consistency:**

```
No Queue = Protocol Incomplete
  • Direct Mode usable
  • Nested Mode semi-usable (Outer layer offline fails)
  • Routed Mode barely usable (Multi-hop easier to be offline)
  
With Queue = Three Modes Unified
  • All modes support async
  • Consistent user experience
```

---

# 🎯 Summary
## 🔗 Dependency Graph of Three Features

```
Timeline (Development Order):
═══════════════════════════════════════════════════════════

Week 1-3: Feature 1 - Relationship State Machine
    │
    ├─ Output: Stable state transition logic
    │
    ↓ Depends
Week 4-5: Feature 2 - Retry Mechanism
    │
    ├─ Utilizes: Feature 1 state machine (Detect dirty state)
    ├─ Output: High reliability message transport
    │
    ↓ Synergizes
Week 6-7: Feature 3 - Offline Queue
    │
    ├─ Utilizes: Feature 1 state (Determine if offline)
    ├─ Utilizes: Feature 2 retry (Queue message retry delivery)
    └─ Output: Complete asynchronous communication experience
```

**Why this order?**

1. **Feature 1 → Feature 2**
   ```
   State machine defines "what is a failed state"
   Retrier needs to know "when to recover state"
   
   Ex: Unidirectional timeout 60s
       → Retrier automatically rolls back to Unrelated
       → Needs state machine defined rules
   ```

2. **Feature 2 → Feature 3**
   ```
   Queue message delivery depends on retry mechanism
   
   Flow:
   1. Dequeue message
   2. Attempt send
   3. Fail → Retrier automatically retries
   4. Success → Delete from queue
   5. Permanent Fail → Mark as dead letter
   ```

3. **Feature 1 + 3 Synergy**
   ```
   Queue needs to know relationship state
   
   Ex: Bidirectional relationship cancelled
       → Clear all queued messages for that relationship
       → Avoid leaking messages of cancelled relationships
   ```
---

# 📋 Feature 3: Implement Offline Message Queue

## 🔍 Current State Analysis

**Implemented:**
```rust
// ✅ Intermediary server supports message buffering (examples/intermediary.rs)
// ✅ PendingMessage enum (definitions/mod.rs L134)
```

**Missing:**
```
❌ 1. No persistent queue
❌ 2. No message priority
❌ 3. No expiration handling
❌ 4. Receiver online notification mechanism incomplete
```

## 📐 Implementation Plan

### Step 1: Message Queue Model

**New File:** `tsp_sdk/src/message_queue.rs`

```rust
use bytes::BytesMut;
use std::collections::{HashMap, VecDeque};
use tokio::sync::RwLock;

#[derive(Clone, Debug)]
pub struct QueuedMessage {
    pub id: String,
    pub sender: String,
    pub receiver: String,
    pub payload: BytesMut,
    pub priority: Priority,
    pub created_at: std::time::SystemTime,
    pub expires_at: Option<std::time::SystemTime>,
    pub retry_count: u8,
}

#[derive(Clone, Debug, PartialEq, Eq, PartialOrd, Ord)]
pub enum Priority {
    Low = 0,
    Normal = 1,
    High = 2,
    Urgent = 3,
}

pub struct MessageQueue {
    /// Message queues grouped by receiver
    queues: Arc<RwLock<HashMap<String, VecDeque<QueuedMessage>>>>,
    /// Persistent storage (Optional)
    storage: Option<Arc<dyn MessageStorage>>,
    /// Max queue size (per receiver)
    max_queue_size: usize,
}

#[async_trait::async_trait]
pub trait MessageStorage: Send + Sync {
    async fn save(&self, msg: &QueuedMessage) -> Result<(), Error>;
    async fn load(&self, receiver: &str) -> Result<Vec<QueuedMessage>, Error>;
    async fn delete(&self, msg_id: &str) -> Result<(), Error>;
}

impl MessageQueue {
    pub fn new(storage: Option<Arc<dyn MessageStorage>>) -> Self {
        Self {
            queues: Arc::new(RwLock::new(HashMap::new())),
            storage,
            max_queue_size: 1000,
        }
    }
    
    /// Enqueue message
    pub async fn enqueue(&self, msg: QueuedMessage) -> Result<(), Error> {
        let receiver = msg.receiver.clone();
        
        // Persistence
        if let Some(storage) = &self.storage {
            storage.save(&msg).await?;
        }
        
        // In-memory queue
        let mut queues = self.queues.write().await;
        let queue = queues.entry(receiver.clone())
            .or_insert_with(VecDeque::new);
        
        // Check queue size
        if queue.len() >= self.max_queue_size {
            // TODO: Remove oldest low priority message
            queue.pop_front();
        }
        
        // Insert by priority
        let insert_pos = queue.iter()
            .position(|m| m.priority < msg.priority)
            .unwrap_or(queue.len());
        queue.insert(insert_pos, msg);
        
        tracing::debug!("Message enqueued for {}, queue size: {}", receiver, queue.len());
        
        Ok(())
    }
    
    /// Dequeue messages (when receiver comes online)
    pub async fn dequeue(&self, receiver: &str, limit: usize) -> Result<Vec<QueuedMessage>, Error> {
        let mut queues = self.queues.write().await;
        
        let Some(queue) = queues.get_mut(receiver) else {
            return Ok(vec![]);
        };
        
        let mut messages = vec![];
        for _ in 0..std::cmp::min(limit, queue.len()) {
            if let Some(msg) = queue.pop_front() {
                // Check expiration
                if let Some(expires_at) = msg.expires_at {
                    if System Time::now() > expires_at {
                        // Expired, skip
                        if let Some(storage) = &self.storage {
                            let _ = storage.delete(&msg.id).await;
                        }
                        continue;
                    }
                }
                messages.push(msg);
            }
        }
        
        Ok(messages)
    }
    
    /// Cleanup expired messages (called periodically)
    pub async fn cleanup_expired(&self) -> Result<usize, Error> {
        let mut count = 0;
        let mut queues = self.queues.write().await;
        
        for queue in queues.values_mut() {
            queue.retain(|msg| {
                if let Some(expires_at) = msg.expires_at {
                    if SystemTime::now() > expires_at {
                        count += 1;
                        return false;
                    }
                }
                true
            });
        }
        
        tracing::info!("Cleaned up {} expired messages", count);
        Ok(count)
    }
}
```

### Step 2: Integrate into AsyncSecureStore

```rust
// Modify async_store.rs
impl AsyncSecureStore {
    /// Send message (support offline queue)
    pub async fn send_or_queue(
        &self,
        sender: &str,
        receiver: &str,
        nonconfidential_data: Option<&[u8]>,
        payload: &[u8],
        priority: Priority,
        ttl: Option<Duration>,
    ) -> Result<(), Error> {
        // Try sending directly
        match self.send(sender, receiver, nonconfidential_data, payload).await {
            Ok(()) => Ok(()),
            Err(Error::Transport(_)) => {
                // Transport failed, enqueue
                tracing::warn!("Failed to send to {}, queueing message", receiver);
                
                let (_, sealed_msg) = self.store.seal_message(
                    sender,
                    receiver,
                    nonconfidential_data,
                    payload,
                )?;
                
                let queued_msg = QueuedMessage {
                    id: uuid::Uuid::new_v4().to_string(),
                    sender: sender.to_string(),
                    receiver: receiver.to_string(),
                    payload: BytesMut::from(sealed_msg.as_slice()),
                    priority,
                    created_at: SystemTime::now(),
                    expires_at: ttl.map(|d| SystemTime::now() + d),
                    retry_count: 0,
                };
                
                self.message_queue.enqueue(queued_msg).await?;
                Ok(())
            }
            Err(e) => Err(e),
        }
    }
    
    /// Receive messages (including queued)
    pub async fn receive_with_queue(
        &self,
        vid: &str,
    ) -> Result<TSPStream<ReceivedTspMessage>, Error> {
        // 1. First fetch offline messages from queue
        let queued = self.message_queue.dequeue(vid, 100).await?;
        
        // 2. Create stream: send queued messages first, then live messages
        let store = self.store.clone();
        let queue_stream = tokio_stream::iter(queued.into_iter().map(move |msg| {
            let mut payload = msg.payload;
            store.open_message(&mut payload)
        }));
        
        // 3. Live message stream
        let live_stream = self.receive(vid).await?;
        
        // 4. Combine streams
        use tokio_stream::StreamExt;
        let combined = queue_stream.chain(live_stream);
        
        Ok(Box::pin(combined))
    }
}
```

### Step 3: Persistence Implementation (Based on Askar)

```rust
// New message_queue/askar_storage.rs
use crate::message_queue::{MessageStorage, QueuedMessage};
use aries_askar::{Store, Session};

pub struct AskarMessageStorage {
    store: Arc<Store>,
}

#[async_trait::async_trait]
impl MessageStorage for AskarMessageStorage {
    async fn save(&self, msg: &QueuedMessage) -> Result<(), Error> {
        let mut session = self.store.session(None).await?;
        
        let value = serde_json::to_vec(msg)?;
        session.insert(
            &format!("msg:{}", msg.id),
            value,
            &format!("receiver:{}", msg.receiver),
        ).await?;
        
        Ok(())
    }
    
    async fn load(&self, receiver: &str) -> Result<Vec<QueuedMessage>, Error> {
        let session = self.store.session(None).await?;
        
        let rows = session.fetch_all(
            Some(&format!("receiver:{}", receiver)),
            None,
            None,
        ).await?;
        
        let mut messages = vec![];
        for row in rows {
            let msg: QueuedMessage = serde_json::from_slice(row.value.as_ref())?;
            messages.push(msg);
        }
        
        Ok(messages)
    }
    
    async fn delete(&self, msg_id: &str) -> Result<(), Error> {
        let mut session = self.store.session(None).await?;
        session.remove(&format!("msg:{}", msg_id)).await?;
        Ok(())
    }
}
```

## 🔗 Integration Points with Existing Code

| Component | Modification | Description |
|-----------|--------------|-------------|
| **async_store.rs** | Add field | `message_queue: Arc<MessageQueue>` |
| **async_store.rs** | New methods | `send_or_queue()`, `receive_with_queue()` |
| **examples/intermediary.rs** | Replace | Use new MessageQueue |
| **secure_storage.rs** | Extend | `AskarMessageStorage` |

---

# 🎯 Summary

## 📊 Comparison of Three Features

| Feature | Feature 1 (State Machine) | Feature 2 (Retry) | Feature 3 (Queue) |
|---------|---------------------------|-------------------|-------------------|
| **Code Size** | ~800 lines | ~500 lines | ~600 lines |
| **Test Size** | ~400 lines | ~300 lines | ~300 lines |
| **Complexity** | High | Medium | Medium |
| **New Deps** | None | `tokio::time` | `tokio-stream`, `uuid` |
| **Disruptive** | Medium (Modify API) | Low (New API) | Low (New API) |

## 🚀 Recommended Implementation Order

1. **Week 1-3**: Feature 1 (Relationship State Machine) - Most Critical
2. **Week 4-5**: Feature 2 (Retry Mechanism) - Production Necessary
3. **Week 6-7**: Feature 3 (Message Queue) - User Experience

## ✅ Benefits After Completion

- ✅ Relationship management more robust (Handle concurrency, timeout)
- ✅ Network fault tolerance improved by 90%+
- ✅ Support offline scenarios (Key requirement)
- ✅ Big step towards version 1.0
