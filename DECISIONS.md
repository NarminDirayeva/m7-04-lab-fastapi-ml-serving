# Architectural Decisions — Vision Moderation Service

## 1. API Versioning Strategy
We have adopted **Path-Based Versioning** (`/v1/`) instead of Header-Based Versioning (`X-API-Version`) based on three strategic pillars:

* **Edge Routing Efficiency:** Path-based versioning enables the API Gateway (e.g., Envoy, Kong) to perform static prefix routing instantly. This decouples routing from intense header parsing overhead, ensuring we meet our strict **p95 latency target of 300 ms**.
* **Enterprise Security & Perimeter Defense:** Web Application Firewalls (WAF) and perimeter security layers inspect URL paths by default. Path routing simplifies the configuration of rate-limiting rules per version at the edge.
* **Developer Ergonomics:** It prevents client integration bugs caused by custom HTTP client configurations or proxy layers that strip out custom headers during upstream transmission.

---

## 2. Batch Processing & Partial Failure Model
For the `/v1/predict-batch` endpoint (which accepts up to 32 images), we rejected the "All-or-Nothing" atomic approach and chose a **Map-Keyed Partial Failure Model** returning an HTTP `200 OK`. 

* **Blast Radius Mitigation:** If 1 out of 32 images is corrupt, rejecting the entire payload with an HTTP `400` or `422` forces the client to retry the entire batch. At our peak traffic of **200 RPS**, this creates a critical retry-storm that could degrade downstream model performance.
* **Payload Structure:** The response map matches the caller-supplied IDs directly to either a successful `PredictResponse` or a specific `Error` schema. 
* **Throughput Optimization:** This guarantees that valid image classifications are processed and delivered without dynamic latency blocking, keeping pipeline multi-tenancy highly isolated and performant.

---

## 3. Asynchronous Job Lifecycle & Retention Policy
The asynchronous architecture for larger image assets relies on an explicit **State-Driven Polymorphism** workflow coupled with a strict data cleanup mechanism.

### State Machine Transitions
An asynchronous job progresses through a strict, non-reversible lifecycle: