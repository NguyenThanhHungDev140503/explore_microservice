# Chapter 2: How to Model Microservices

## Subsections
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

## Summary
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
