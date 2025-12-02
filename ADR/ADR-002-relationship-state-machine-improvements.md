# 🏆 Feature 1: Relationship State Machine Improvements

## ✅ Why is this the highest priority?

### 1️⃣ **Core Semantics of TSP Protocol**

As seen in the technical specification document:

```
TSP-technical-specification.md (L116):
"S2: The SDK will support 'direct mode', 'nested mode' and 'routed mode'"
"Some design discussion around control messages setting up 
 routed mode will be needed in the future."
```

**Key Points:**
- Relationship management is the foundation of the **three major modes** of TSP.
- Direct Mode, Nested Mode, and Routed Mode all rely on correct relationship states.
- **No robust relationship state machine = TSP is incomplete.**

### 2️⃣ **Contradictory State of Current Code**

A serious issue was found in the code:

```rust
// Defined states (definitions/mod.rs L48)
pub enum RelationshipStatus {
    _Controlled,
    Bidirectional { thread_id, outstanding_nested_thread_ids },
    Unidirectional { thread_id },
    ReverseUnidirectional { thread_id },  // ← Defined but rarely used!
    Unrelated,
}

// Actual usage (grep results)
Unidirectional: Used 19 times
ReverseUnidirectional: Used 3 times  // ❌ Seriously insufficient!
```

**Problem Diagnosis:**

```rust
// store.rs L740 - Receiving a relationship request
Payload::RequestRelationship { route, thread_id } => {
    // ❌ Problem: No check for current state!
    Ok(ReceivedTspMessage::RequestRelationship { 
        sender,
        receiver: intended_receiver,
        route,
        thread_id,
        nested_vid: None,
    })
}

// What should be done?
// 1. Check if a relationship already exists
// 2. If Unidirectional, handle concurrency
// 3. If Bidirectional, reject duplicate requests
// 4. Transition to ReverseUnidirectional state
```

**Real-world Scenarios:**

```
Scenario 1: Concurrent Relationship Requests
────────────────────────────────────
Alice                    Bob
  │                       │
  ├──Request───────────→  │  (Alice -> Unidirectional)
  │  ←──────────Request──┤  (Bob -> Unidirectional)
  │                       │
  Result: Both sides are Unidirectional!
  ❌ Who should accept? Who should reject?
  ❌ Current code does not handle this!
```

```
Scenario 2: Request Timeout
────────────────────────────────────
Alice                    Bob (Offline)
  │                       │
  ├──Request───────────→  ✗
  │                       
  (Alice enters Unidirectional state)
  
  60 seconds later... Bob is still offline
  ❌ Alice should timeout and revert to Unrelated
  ❌ Current code will wait forever!
```

```
Scenario 3: Duplicate Requests (Packet Loss)
────────────────────────────────────
Alice                    Bob
  │                       │
  ├──Request(1)────────→  │ (Received)
  │                       │
  ├──Request(2)────────→  │ (Retransmission)
  │                       │
  
  ❌ Bob will process it twice!
  ❌ Should be idempotent, but currently is not
```

### 3️⃣ **Huge Impact Scope**

Relationship state affects almost all operations:

```rust
// Affected functional modules
┌─────────────────────────────────────┐
│ Relationship State Machine          │
├─────────────────────────────────────┤
│ ↓ Affects                           │
│ • seal_message() - Must have relationship to encrypt
│ • make_nested_relationship_request()│
│ • make_new_identifier_notice()      │
│ • route_message() - Routing needs relationship
│ • forward_routed_message()          │
│ • All control messages              │
└─────────────────────────────────────┘

// If the state machine has bugs:
❌ User A and B states are inconsistent →
❌ A thinks there is a relationship, B thinks not →
❌ A sends a message, B rejects it →
❌ User experience collapses
```

### 4️⃣ **Code has 60% Foundation**

This is the best part:

```rust
// ✅ Existing foundation (can be leveraged)
1. RelationshipStatus enum fully defined
2. Payload types for all control messages
3. Framework for make_relationship_request/accept/cancel
4. Test case framework (L1389-1504)

// ❌ Missing (we need to add)
1. State transition rules (~200 lines)
2. Concurrency handling logic (~150 lines)
3. Timeout mechanism (~100 lines)
4. Complete tests (~400 lines)

Total: ~850 lines, but based on existing 2000+ lines of code
Input/Output Ratio: 1:2.3
```

# 📋 Feature 1: Implement Relationship State Machine

## 🔍 Current State Analysis

**Implemented:**
```rust
// ✅ RelationshipStatus enum definition complete
pub enum RelationshipStatus {
    _Controlled,                    // Own VID
    Bidirectional { thread_id, .. },// Bidirectional relationship
    Unidirectional { thread_id },   // Unidirectional (I initiated)
    ReverseUnidirectional { thread_id }, // Reverse Unidirectional (Other initiated)
    Unrelated,                      // No relationship
}

// ✅ Basic control messages
- RequestRelationship（L947）
- AcceptRelationship（L984）
- CancelRelationship（L1029）

// ✅ Nested relationship support
- RequestNestedRelationship（L1053）
- AcceptNestedRelationship（L1100）
```

**Missing or Incomplete:**
```rust
// ❌ 1. No clear state transition diagram
// ❌ 2. Incomplete handling of edge cases (duplicate requests, timeouts, etc.)
// ❌ 3. Recovery mechanism for inconsistent states
// ❌ 4. ReverseUnidirectional state not fully utilized
// ❌ 5. Lack of detailed logs for state transitions
// ❌ 6. No concurrent request handling strategy
```

## 📐 Implementation Plan

### Step 1: Define Complete State Transition Table

**New File:** `tsp_sdk/src/relationship_machine.rs`

```rust
// State machine definition
use crate::definitions::{Digest, RelationshipStatus};

/// Relationship State Machine Events
#[derive(Debug, Clone)]
pub enum RelationshipEvent {
    // Locally initiated
    SendRequest { thread_id: Digest },
    SendAccept { thread_id: Digest },
    SendCancel,
    
    // Received
    ReceiveRequest { thread_id: Digest },
    ReceiveAccept { thread_id: Digest },
    ReceiveCancel,
    
   // Timeout/Error
    Timeout,
    Error(String),
}

/// State Transition Rules
pub struct RelationshipMachine;

impl RelationshipMachine {
    /// Core state transition logic
    pub fn transition(
        current: RelationshipStatus,
        event: RelationshipEvent,
    ) -> Result<RelationshipStatus, StateError> {
        match (current, event) {
            // Unrelated -> Unidirectional (I initiate request)
            (RelationshipStatus::Unrelated, 
             RelationshipEvent::SendRequest { thread_id }) => {
                Ok(RelationshipStatus::Unidirectional { thread_id })
            }
            
            // Unrelated -> ReverseUnidirectional (Receive request)
            (RelationshipStatus::Unrelated, 
             RelationshipEvent::ReceiveRequest { thread_id }) => {
                Ok(RelationshipStatus::ReverseUnidirectional { thread_id })
            }
            
            // Unidirectional -> Bidirectional (Receive Accept)
            (RelationshipStatus::Unidirectional { thread_id: my_id }, 
             RelationshipEvent::ReceiveAccept { thread_id }) => {
                if my_id == thread_id {
                    Ok(RelationshipStatus::Bidirectional { 
                        thread_id,
                        outstanding_nested_thread_ids: vec![],
                    })
                } else {
                    Err(StateError::ThreadIdMismatch)
                }
            }
            
            // ReverseUnidirectional -> Bidirectional (Send Accept)
            (RelationshipStatus::ReverseUnidirectional { thread_id },
             RelationshipEvent::SendAccept { thread_id: their_id }) => {
                if thread_id == their_id {
                    Ok(RelationshipStatus::Bidirectional {
                        thread_id,
                        outstanding_nested_thread_ids: vec![],
                    })
                } else {
                    Err(StateError::ThreadIdMismatch)
                }
            }
            
            // Bidirectional -> Unrelated (Any side cancels)
            (RelationshipStatus::Bidirectional { .. },
             RelationshipEvent::SendCancel | RelationshipEvent::ReceiveCancel) => {
                Ok(RelationshipStatus::Unrelated)
            }
            
            // Handle duplicate requests
            (RelationshipStatus::Unidirectional { thread_id },
             RelationshipEvent::SendRequest { .. }) => {
                // Idempotent: return current state
                Ok(RelationshipStatus::Unidirectional { thread_id })
            }
            
            // Invalid transition
            (current, event) => {
                Err(StateError::InvalidTransition { 
                    from: current, 
                    event 
                })
            }
        }
    }
    
    /// Verify if state allows an operation
    pub fn can_send_message(status: &RelationshipStatus) -> bool {
        matches!(status, 
            RelationshipStatus::Bidirectional { .. } | 
            RelationshipStatus::_Controlled)
    }
    
    /// Get allowed next events
    pub fn allowed_events(status: &RelationshipStatus) -> Vec<RelationshipEvent> {
        match status {
            RelationshipStatus::Unrelated => vec![
                RelationshipEvent::SendRequest { thread_id: [0; 32] },
                RelationshipEvent::ReceiveRequest { thread_id: [0; 32] },
            ],
            RelationshipStatus::Unidirectional { .. } => vec![
                RelationshipEvent::ReceiveAccept { thread_id: [0; 32] },
                RelationshipEvent::SendCancel,
                RelationshipEvent::Timeout,
            ],
            // ... other states
            _ => vec![],
        }
    }
}

#[derive(Debug, thiserror::Error)]
pub enum StateError {
    #[error("Invalid state transition from {from:?} with event {event:?}")]
    InvalidTransition {
        from: RelationshipStatus,
        event: RelationshipEvent,
    },
    #[error("Thread ID mismatch")]
    ThreadIdMismatch,
}
```

**Integration into store.rs:**

```rust
// Modify store.rs (around L930-1033)
impl SecureStore {
    pub fn make_relationship_request(
        &self,
        sender: &str,
        receiver: &str,
        route: Option<&[&str]>,
    ) -> Result<(Url, Vec<u8>), Error> {
        // New: Check current state
        let current_status = self.relation_status_for_vid_pair(sender, receiver)?;
        
        // New: State machine validation
        let mut thread_id = Default::default();
        let event = RelationshipEvent::SendRequest { thread_id };
        let new_status = RelationshipMachine::transition(current_status, event)?;
        
        // Original logic...
        let tsp_message = crate::crypto::seal_and_hash(...)?;
        
        // New: Log state transition
        tracing::info!(
            "Relationship state: {} -> {}", 
            current_status, 
            new_status
        );
        
        // Update state
        self.set_relation_and_status_for_vid(receiver, new_status, sender)?;
        
        Ok((transport, tsp_message.to_owned()))
    }
}
```

### Step 2: Add Timeout and Retransmission Mechanism

**Add fields to VidContext:**

```rust
// Modify store.rs (L23-31)
struct VidContext {
    vid: Arc<dyn VerifiedVid>,
    private: Option<Arc<dyn PrivateVid>>,
    parent_vid: Option<String>,
    relation_vid: Option<String>,
    route: Option<Vec<String>>,
    relation_status: RelationshipStatus,
    metadata: Option<serde_json::Value>,
    
    // New fields ↓
    /// Timeout for relationship request
    request_timeout: Option<std::time::Instant>,
    /// Pending request (for retransmission)
    pending_request: Option<PendingRequest>,
}

#[derive(Clone, Debug)]
struct PendingRequest {
    thread_id: Digest,
    sent_at: std::time::Instant,
    retry_count: u8,
    max_retries: u8,
}
```

**Add timeout check method:**

```rust
impl SecureStore {
    /// Check and handle timed-out relationship requests
    pub fn check_timeouts(&self) -> Result<Vec<TimeoutAction>, Error> {
        let mut actions = vec![];
        let mut vids = self.vids.write()?;
        
        for (vid, context) in vids.iter_mut() {
            if let Some(timeout) = context.request_timeout {
                if timeout.elapsed() > Duration::from_secs(60) {
                    // Handle timeout
                    if let Some(ref mut pending) = context.pending_request {
                        if pending.retry_count < pending.max_retries {
                            // Retry
                            pending.retry_count += 1;
                            pending.sent_at = Instant::now();
                            actions.push(TimeoutAction::Retry {
                                vid: vid.clone(),
                                thread_id: pending.thread_id,
                            });
                        } else {
                            // Give up, rollback state
                            context.relation_status = RelationshipStatus::Unrelated;
                            context.request_timeout = None;
                            context.pending_request = None;
                            actions.push(TimeoutAction::Fail {
                                vid: vid.clone(),
                            });
                        }
                    }
                }
            }
        }
        
        Ok(actions)
    }
}

pub enum TimeoutAction {
    Retry { vid: String, thread_id: Digest },
    Fail { vid: String },
}
```

### Step 3: Concurrent Request Handling

```rust
impl SecureStore {
    /// Handle received relationship request (handle concurrency)
    pub fn handle_relationship_request(
        &self,
        my_vid: &str,
        their_vid: &str,
        their_thread_id: Digest,
    ) -> Result<RequestAction, Error> {
        let current_status = self.relation_status_for_vid_pair(my_vid, their_vid)?;
        
        match current_status {
            // Normal case: No relationship
            RelationshipStatus::Unrelated => {
                let new_status = RelationshipStatus::ReverseUnidirectional {
                    thread_id: their_thread_id,
                };
                self.set_relation_and_status_for_vid(their_vid, new_status, my_vid)?;
                Ok(RequestAction::Accept)
            }
            
            // Concurrent conflict: Both sides initiated simultaneously
            RelationshipStatus::Unidirectional { thread_id: my_thread_id } => {
                // Use thread_id sorting to decide whose request has priority
                if my_thread_id < their_thread_id {
                    // My request has priority, reject theirs
                    Ok(RequestAction::Reject {
                        reason: "Concurrent request, mine has priority".into(),
                    })
                } else {
                    // Their request has priority, accept and cancel mine
                    self.cancel_pending_request(my_vid, their_vid)?;
                    let new_status = RelationshipStatus::ReverseUnidirectional {
                        thread_id: their_thread_id,
                    };
                    self.set_relation_and_status_for_vid(their_vid, new_status, my_vid)?;
                    Ok(RequestAction::Accept)
                }
            }
            
            // Existing relationship: Idempotent handling
            RelationshipStatus::ReverseUnidirectional { thread_id } |
            Relationship Status::Bidirectional { thread_id, .. } => {
                if thread_id == their_thread_id {
                    // Duplicate request, idempotent response
                    Ok(RequestAction::AlreadyHandled)
                } else {
                    // New request, reject
                    Ok(RequestAction::Reject {
                        reason: "Already in a relationship".into(),
                    })
                }
            }
            
            _ => Err(Error::Relationship("Invalid state".into())),
        }
    }
    
    fn cancel_pending_request(&self, my_vid: &str, their_vid: &str) -> Result<(), Error> {
        let mut vids = self.vids.write()?;
        if let Some(context) = vids.get_mut(their_vid) {
            context.pending_request = None;
            context.request_timeout = None;
        }
        Ok(())
    }
}

pub enum RequestAction {
    Accept,
    Reject { reason: String },
    AlreadyHandled,
}
```

### Step 4: Test Suite

**New Test File:** `tsp_sdk/src/relationship_tests.rs`

```rust
#[cfg(test)]
mod relationship_state_tests {
    use super::*;
    
    #[test]
    fn test_state_machine_normal_flow() {
        // Unrelated -> Unidirectional -> Bidirectional
        let initial = RelationshipStatus::Unrelated;
        let thread_id = [1u8; 32];
        
        // Send request
        let state1 = RelationshipMachine::transition(
            initial,
            RelationshipEvent::SendRequest { thread_id },
        ).unwrap();
        assert!(matches!(state1, RelationshipStatus::Unidirectional { .. }));
        
        // Receive accept
        let state2 = RelationshipMachine::transition(
            state1,
            RelationshipEvent::ReceiveAccept { thread_id },
        ).unwrap();
        assert!(matches!(state2, RelationshipStatus::Bidirectional { .. }));
    }
    
    #[test]
    fn test_concurrent_requests() {
        let store = SecureStore::new();
        let alice = OwnedVid::new_did_peer("tcp://alice".parse().unwrap());
        let bob = OwnedVid::new_did_peer("tcp://bob".parse().unwrap());
        
        store.add_private_vid(alice.clone(), None).unwrap();
        store.add_private_vid(bob.clone(), None).unwrap();
        
        // Alice sends request
        let (_, msg1) = store.make_relationship_request(
            alice.identifier(),
            bob.identifier(),
            None,
        ).unwrap();
        
        // Bob sends request simultaneously (concurrent)
        let (_, msg2) = store.make_relationship_request(
            bob.identifier(),
            alice.identifier(),
            None,
        ).unwrap();
        
        // Simulate receiving (one should be rejected)
        let result1 = store.open_message(&mut msg1.clone());
        let result2 = store.open_message(&mut msg2.clone());
        
        // Verify only one relationship is successfully established
        // Specific logic...
    }
    
    #[test]
    fn test_timeout_and_retry() {
        // Test timeout retry logic
    }
    
    #[test]
    fn test_idempotent_requests() {
        // Test idempotency of duplicate requests
    }
}
```

## 🔗 Integration Points with Existing Code

| Component | Modification Location | New Content |
|-----------|-----------------------|-------------|
| **definitions/mod.rs** | Add | `RelationshipEvent` enum |
| **store.rs** | Modify L930-1033 | State machine call logic |
| **store.rs** | Modify L23-31 | `VidContext` new fields |
| **error.rs** | Add | `StateError` error type |
| **async_store.rs** | Add method | `check_timeouts_async()` |
