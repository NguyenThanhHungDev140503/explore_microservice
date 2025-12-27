# Chapter 4: Microservice Communication Styles

## Subsections
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

## Summary
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

## Code Examples

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
