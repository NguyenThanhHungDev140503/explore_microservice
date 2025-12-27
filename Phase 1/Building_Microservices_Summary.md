# Building Microservices: Designing Fine-Grained Systems (2nd Edition) - Summary

## Chapter 1: What Are Microservices?

### Subsections
*   **Microservices at a Glance**
*   **Key Concepts of Microservices**
    *   Independent Deployability
    *   Modeled Around a Business Domain
    *   Owning Their Own State
    *   Size
    *   Flexibility
    *   Alignment of Architecture and Organization
*   **The Monolith**
*   **The Enabling Technology**
    *   Log Aggregation and Distributed Tracing
    *   Containers and Kubernetes
    *   Streaming
    *   Cloud Public
*   **Advantages**
    *   Technology Heterogeneity
    *   Robustness
    *   Scaling
    *   Ease of Deployment
    *   Organizational Alignment
    *   Composability
*   **Pain Points**
    *   Developer Experience
    *   Technology Overload
    *   Cost
    *   Reporting
    *   Monitoring
    *   Security
    *   Testing
    *   Latency
    *   Data Consistency
*   **Who Is This Book For?**

### Summary
This chapter introduces the concept of microservices as independently releasable services modeled around a business domain. It contrasts them with Service-Oriented Architecture (SOA), positioning microservices as a specific, opinionated approach to SOA with a strong focus on independent deployability.

Key concepts include:
*   **Independent Deployability:** The golden rule. You should be able to deploy a change to a single microservice without touching others. This drives loose coupling and stable contracts.
*   **Domain Modeling:** Services should represent business domains (e.g., Inventory, Order Management) rather than technical layers, utilizing Domain-Driven Design (DDD) principles.
*   **State Ownership:** Services encapsulates their own data. Shared databases are discouraged to prevent coupling.
*   **Size:** "Micro" is misleading. Size should be manageable ("as big as your head") and interfaces should be small. The focus is on boundaries, not lines of code.

The author acknowledges usage of the "Monolith" isn't inherently bad but highlights its limitations (coupling, scaling difficulties). Key enabling technologies like containers (Docker/Kubernetes), log aggregation, and public clouds have fueled microservice adoption.

**Advantages:**
*   **Technology Heterogeneity:** Use the right tool for the job (e.g., Python for data, Go for performance).
*   **Robustness:** Failure in one service doesn't necessarily take down the whole system (bulkheading).
*   **Scaling:** Scale only the parts that need it, optimizing resource usage.
*   **Organizational Alignment:** Teams can own vertical slices of functionality, reducing handoffs (Conway's Law).

**Pain Points:**
*   **Complexity:** Distributed systems introduce new failure modes (network latency, partial failures).
*   **Data Consistency:** Distributed transactions are hard; eventual consistency is often required.
*   **Operational Overhead:** More moving parts mean more monitoring, deployment complexity, and infrastructure costs.

### Code Examples
*No specific code examples were provided in this introductory chapter.*

---

## Chapter 2: How to Model Microservices

### Subsections
*   **Introducing MusicCorp**
*   **What Makes a Good Microservice Boundary?**
    *   Information Hiding
    *   Cohesion
    *   Coupling
*   **Coupling and Cohesion**
*   **Types of Coupling**
    *   Domain Coupling
    *   Pass-Through Coupling
    *   Common Coupling
    *   Content Coupling
*   **Just Enough Domain-Driven Design**
    *   Ubiquitous Language
    *   Aggregate
    *   Bounded Context
    *   Mapping Aggregates and Bounded Contexts to Microservices
    *   Event Storming
*   **Alternatives to Business Domain Boundaries**
    *   Volatility
    *   Data
    *   Technology
    *   Organizational
*   **Mixing Models and Exceptions**

### Summary
This chapter focuses on defining service boundaries, arguably the most critical and difficult part of microservice architecture. It emphasizes **Information Hiding**, **High Cohesion**, and **Low Coupling**.

*   **Cohesion:** Code that changes together stays together. Related functionality should be grouped.
*   **Coupling:** A measure of how dependent one thing is on another. The goal is loose coupling so changes in one service don't cascade.

**Types of Coupling (from loose to tight):**
1.  **Domain Coupling:** Use of another service's functionality. Unavoidable but should be minimized.
2.  **Pass-Through Coupling:** A service passes data it doesn't need to another service.
3.  **Common Coupling:** Services share a common data source (like a shared database). Dangerous.
4.  **Content Coupling:** One service modifies another's internal state directly. The worst form.

The author advocates for **Domain-Driven Design (DDD)** as the primary tool for finding boundaries:
*   **Bounded Contexts:** Explicit boundaries within which a domain model applies. These often map 1:1 to microservices.
*   **Aggregates:** Clusters of domain objects treated as a unit (e.g., an Order and its Line Items). These shouldn't span microservices.

Alternative decomposition strategies (Volatility, Data, Technology) are mentioned but treated as secondary to domain-based decomposition.

### Code Examples
*No specific code examples were provided in this chapter, but architectural diagrams (Figures 2-X) are heavily referenced to illustrate coupling types.*

---

## Chapter 3: Splitting the Monolith

### Subsections
*   **Have a Goal**
*   **Incremental Migration**
*   **The Monolith Is Rarely the Enemy**
*   **The Dangers of Premature Decomposition**
*   **What to Split First?**
*   **Decomposition by Layer**
    *   Code First
    *   Data First
*   **Useful Decompositional Patterns**
    *   Strangler Fig Pattern
    *   Parallel Run
    *   Feature Toggle
*   **Data Decomposition Concerns**
    *   Performance
    *   Data Integrity
    *   Transactions
    *   Tooling
    *   Reporting Database

### Summary
This chapter guides the migration from a monolith to microservices. The refrain is **"Incremental Migration"**—chip away at the monolith rather than attempting a "Big Bang" rewrite.

**Key Strategies:**
*   **Strangler Fig Pattern:** Gradually replace functionality by routing calls to new microservices around the edges of the monolith, eventually choking off the old system.
*   **Parallel Run:** Run old and new implementations side-by-side to verify correctness before switching.
*   **Feature Toggles:** Control the rollout of new microservices.

**Data Decomposition** is highlighted as the hardest part:
*   **Performance:** Joining data across services (application-side joins) is slower than database joins.
*   **Integrity:** You lose foreign key constraints across services.
*   **Transactions:** You lose ACID transactions spanning multiple services. Distributed transactions are complex; alternatives like Sagas are preferred.
*   **Reporting:** A separate reporting database (populated via events or batch jobs) is often needed to handle querying needs without coupling services.

**Advice:**
*   Don't split just for the sake of it.
*   Start with low-hanging fruit or areas that need independent scaling/deployment.
*   Premature decomposition (before understanding the domain) is costly.

### Code Examples
*No specific code examples were provided in this chapter.*

---

## Chapter 4: Microservice Communication Styles

### Subsections
*   **From In-Process to Inter-Process**
    *   Performance
    *   Changing Interfaces
    *   Error Handling
*   **Technology for Inter-Process Communication: So Many Choices**
*   **Styles of Microservice Communication**
*   **Pattern: Synchronous Blocking**
    *   Advantages
    *   Disadvantages
    *   Where to Use It
*   **Pattern: Asynchronous Nonblocking**
    *   Advantages
    *   Disadvantages
    *   Where to Use It
*   **ASYNC/AWAIT AND WHEN ASYNCHRONOUS IS STILL BLOCKING**
*   **Pattern: Communication Through Common Data**
    *   Implementation
    *   Advantages
    *   Disadvantages
    *   Where to Use It
*   **Pattern: Request-Response Communication**
    *   Commands Versus Requests
    *   Implementation: Synchronous Versus Asynchronous
    *   Parallel Versus Sequential Calls
    *   Where to Use It
*   **Pattern: Event-Driven Communication**
    *   Events and Messages
    *   Implementation
    *   What’s in an Event? (Just an ID vs. Fully Detailed)
    *   Where to Use It
    *   Proceed with Caution

### Summary
This chapter analyzes how microservices talk to each other, emphasizing the trade-offs between different styles.

**Key Distinctions:**
1.  **Synchronous Blocking:** The caller waits for a response (e.g., HTTP GET). Simple and familiar but introduces **temporal coupling** (downstream service must be up) and can lead to cascading failures.
2.  **Asynchronous Nonblocking:** The caller sends a message and continues (e.g., AMQP, Callbacks). Decouples services temporarily but adds complexity (handling responses, correlation IDs).

**Communication Patterns:**
*   **Request-Response:** "Do this for me." Can be sync or async. Good for querying or immediate commands.
*   **Event-Driven:** "This happened." Inverts control. The emitter doesn't know who is listening. Promotes extreme loose coupling but can be hard to trace/debug.
*   **Common Data:** (Data Lakes/Warehouses) Good for bulk data, bad for operational coupling.

**Important Nuance:** `async/await` in code often still results in **blocking** behavior from a system perspective if the operation waits for the result before proceeding.

**Event Design:**
*   **Just an ID:** Small payloads, but requires receivers to call back (coupling, "thundering herd").
*   **Fully Detailed:** Larger payloads, but receivers are self-sufficient (better decoupling).

### Code Examples

**Example 4-1: Working with a potentially asynchronous call in a blocking, synchronous fashion (JavaScript)**
```javascript
async function f() {

  let eurToGbp = new Promise((resolve, reject) => {
    //code to fetch latest exchange rate between EUR and GBP
    ...
  });

  var latestRate = await eurToGbp;
  process(latestRate);
}
```
*Note: Even though the operation is "async", `await` blocks the logical flow until the promise resolves.*
