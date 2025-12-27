# Module 3: Advanced Operations (Service Mesh & Kubernetes)

## 1. Container Management & Orchestration (Kubernetes)
*Based on Chapter 2: Managing Microservices at Scale*

As the number of microservices grows, manual management becomes impossible ("The Microservice Tax"). We need automated systems.

### Containers vs. Virtual Machines
- **VMs**: Heavyweight, full OS. Good for isolation but slow to start.
- **Containers**: Lightweight, shared OS kernel. Fast startup, perfect for stateless event processors.

### Kubernetes Logic for EDA
- **Stateless Consumers**: Can be scaled horizontally (`Replicas`) based on *consumer lag* (not just CPU/Memory). KEDA (Kubernetes Event-driven Autoscaling) is the standard pattern here.
- **Stateful Streaming**: Some frameworks (like Kafka Streams) require state. K8s `StatefulSets` ensure that a pod keeps its identity and persistent storage, which is crucial for local state stores (RocksDB).

## 2. Deployment Patterns for EDA
*Based on Chapter 16: Deploying Event-Driven Microservices*

Deploying EDA services differs from Request/Response services because of *continuous data flow*.

### Full-Stop Deployment
- **Method**: Stop all older consumers, start all new ones.
- **Impact**: Processing pauses. Event lag accumulates in the broker.
- **Pros**: Simplest to implement. No schema compatibility issues (if you change the logic completely).

### Rolling Updates (K8s Default)
- **Method**: Replace pods one by one.
- **Impact**: Zero downtime *if* schema is compatible.
- **Challenge**: Two versions of the consumer (v1 and v2) will be processing events *simultaneously*. This can cause issues if v2 writes a new event format that v1 doesn't understand (if they share an output topic).

### Blue/Green Deployment
- **Method**: Spin up a full V2 cluster alongside V1. Switch over traffic.
- **EDA Nuance**: You can't just "switch traffic" like a Load Balancer. Both V1 and V2 might be consuming from the *same* topic group. You need to manage *Consumer Groups* carefully to avoid processing events twice.

## 3. Service Mesh (Supplementary Content)
*Not explicitly covered in existing book chapters, but critical for modern EDA.*

While the book focuses on "Supportive Tooling" (Chapter 14) like Schema Registries and Offset Managers, **Service Mesh** (Istio, Linkerd) adds a layer of observability and control at the *network* level.

### Why Service Mesh for EDA?
1.  **mTLS**: Securing communication between the Event Broker and the Consumers.
2.  **Observability**: Tracing (Jaeger/Zipkin) is hard in EDA. A mesh can inject trace headers automatically, but *only* for HTTP/gRPC. For Kafka, you typically need specific client instrumentation, as the Mesh acts as a TCP proxy and doesn't "see" individual messages.
3.  **Circuit Breaking**: Protecting the broker from a consumer that is crashing and restarting in a loop.

### Recommendation
For pure Event-Driven systems (Kafka), Service Mesh offers less value than for Request-Response (HTTP) systems. However, in a hybrid architecture (Chapter 13), Service Mesh is vital for the synchronous parts of your system.
