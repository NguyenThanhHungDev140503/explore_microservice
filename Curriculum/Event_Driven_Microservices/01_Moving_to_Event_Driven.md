# Module 1: Moving to Event-Driven Architecture (EDA)

## 1. Core Concepts
*Based on Chapter 1: Why Event-Driven Microservices*

### What are Event-Driven Microservices?
Event-Driven Microservices are a specific architectural style where services communicate primarily through **events** (facts that happened) rather than **requests** (commands to do something). 

### Synchronous vs. Asynchronous Communication
- **Synchronous (Request-Response)**:
    - Service A calls Service B and waits for a response.
    - **Pros**: Simple to reason about, immediate feedback.
    - **Cons**: Temporal coupling (both must be up), cascading failures, latency addition.
- **Asynchronous (Event-Driven)**:
    - Service A emits an event. Service B (and C, D) consumes it when ready.
    - **Pros**: Temporal decoupling, easier scaling, pluggability of new services.
    - **Cons**: Eventual consistency, harder to trace/debug (requires distributed tracing).

### Key Benefits
1.  **Decoupling**: Services don't need to know who consumes their data.
2.  **Scalability**: Producers and Consumers scale independently.
3.  **Resilience**: If a consumer is down, the broker buffers events until it recovers.

## 2. Strategic vs. Tactical Patterns
*Based on Chapter 1 (DDD) and Chapter 2 (Fundamentals)*

### Strategic Patterns (Domain-Driven Design)
- **Bounded Contexts**: Define strict boundaries for each microservice. Events are the "contracts" that cross these boundaries.
- **Context Mapping**: How different contexts relate (e.g., Supplier vs. Customer). In EDA, these relationships are defined by event streams.

### Tactical Patterns (Implementation)
- **Event Structure**:
    - **Notification Events**: "Something changed" (minimal payload, consumer calls back for data - *avoid this in pure EDA if possible*).
    - **Carried State Transfer (Entity Events)**: "Here is the full new state" (fully self-contained, high decoupling).
- **Single Writer Principle**: Only *one* microservice should own writing to a specific stream/topic to ensure consistency.

## 3. Migration and Data Liberation
*Based on Chapter 4: Integrating Event-Driven Architectures with Existing Systems*

How do we move from a Monolith or Legacy System to EDA? We use **Data Liberation**.

### Pattern 1: Liberating Data by Query
- **Mechanism**: A poller service queries the legacy database periodically (e.g., `SELECT * FROM Orders WHERE updated_at > last_check`).
- **Pros**: Non-intrusive, works with any DB.
- **Cons**: Polling overhead, potential for missed updates between polls, "deletes" are hard to track.

### Pattern 2: Change-Data Capture (CDC) Logs
- **Mechanism**: Read the database transaction log (e.g., MySQL Binlog, PostgreSQL WAL) directly.
- **Pros**: Real-time, captures all changes (inserts, updates, deletes), no polling overhead.
- **Cons**: Tight coupling to DB internals, often requires specific tooling (e.g., Debezium).

### Pattern 3: Outbox Pattern
- **Mechanism**: The service updates its entity table AND inserts a row into an `Outbox` table in the *same transaction*. A separate process pushes the Outbox entries to the Event Broker.
- **Pros**: Guarantees "At-least-once" delivery, no dual-write consistency issues.
- **Cons**: Requires modifying the application schema.
