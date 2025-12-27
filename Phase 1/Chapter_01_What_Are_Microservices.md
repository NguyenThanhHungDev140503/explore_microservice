# Chapter 1: What Are Microservices?

## Subsections
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

## Summary
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
