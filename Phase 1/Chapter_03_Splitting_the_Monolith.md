# Chapter 3: Splitting the Monolith

## Subsections
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

## Summary
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
