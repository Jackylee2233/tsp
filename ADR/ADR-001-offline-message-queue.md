# ADR 003: Offline Message Queue

## Status
Proposed

## Context
When sending TSP messages, the transport layer (e.g., TCP, HTTP) may be unavailable, or the recipient may be offline. Currently, if a send fails, the message is lost unless the application manually handles it. We need a mechanism to queue these messages and attempt to resend them later.

## Decision
We will implement an in-memory **Offline Message Queue** within the `SecureStore`.

### 1. `MessageQueue` Structure
We will create a new module `queue.rs` with a `MessageQueue` struct.
- **Storage**: `VecDeque<QueuedMessage>`
- **`QueuedMessage`**:
    - `message`: `Vec<u8>` (The sealed TSP message)
    - `url`: `Url` (The destination)
    - `priority`: `u8` (Optional, for future use)
    - `created_at`: `Instant`

### 2. Integration with `SecureStore`
- `SecureStore` will hold a `Arc<RwLock<MessageQueue>>`.
- **Enqueue**: When a message cannot be sent (e.g., transport error), the application (or `AsyncSecureStore`) can call `store.queue_message(url, message)`.
- **Dequeue/Flush**: A method `store.process_queue()` (or similar) will be available to retrieve messages for attempting to resend.

### 3. Integration with `AsyncSecureStore`
- `AsyncSecureStore` is the active component that handles sending.
- It will check the queue periodically or upon reconnection events.
- When the queue is not empty, it will attempt to send the messages.
- If successful, the message is removed. If failed, it remains (or is moved to the back with a backoff, reusing Feature 2's logic if applicable, though Feature 2 is specific to Relationship Requests).

### 4. Persistence
For this iteration, the queue is **in-memory only**. If the application restarts, queued messages are lost. Persistence (to disk/DB) is out of scope for now but the design should allow for it later (e.g., by serializing `MessageQueue`).

## Consequences
- **Reliability**: Messages are not lost during temporary network outages.
- **Memory Usage**: Queued messages consume memory. We may need a cap on queue size.
- **Ordering**: `VecDeque` preserves FIFO order, which is generally desired.

