# Architectural Decisions — Vision Moderation Service

## 1. API Versioning Strategy
We have adopted **Path-Based Versioning** (`/v1/`) instead of Header-Based Versioning (`X-API-Version`) based on three strategic pillars:

* **Edge Routing Efficiency:** Path-based versioning enables the API Gateway (e.g., Envoy, Kong) to perform static prefix routing instantly. This decouples routing from header parsing overhead and helps meet the strict **p95 latency target of 300 ms**.
* **Enterprise Security & Perimeter Defense:** Web Application Firewalls (WAF) and perimeter security layers inspect URL paths by default. Path routing simplifies rate-limiting and filtering rules at the edge.
* **Developer Ergonomics:** It reduces integration issues caused by misconfigured HTTP clients or intermediaries that may strip custom headers, making the API more predictable and easier to consume.

---

## 2. Batch Processing & Partial Failure Model
For the `/v1/predict-batch` endpoint (accepting up to 32 images), we rejected the all-or-nothing atomic approach and implemented a **Map-Keyed Partial Failure Model** returning an HTTP `200 OK`.

* **Blast Radius Mitigation:** If one image in the batch is corrupt, rejecting the entire request (e.g., `422`) would force clients to retry all items. At peak traffic (**200 RPS**), this can cause retry storms and degrade system stability.
* **Payload Structure:** The response preserves caller-supplied IDs and maps each item to either a successful `PredictResponse` or an `Error` object.
* **Throughput Optimization:** Valid images are processed independently, ensuring maximum throughput and minimizing latency amplification across tenants in a multi-tenant system.

---

## 3. Asynchronous Job Lifecycle & Retention Policy
The asynchronous processing model is designed for larger images or workloads exceeding synchronous latency constraints. It follows a strict **state-driven lifecycle** with explicit transitions and controlled data retention.

### State Machine Transitions
An asynchronous job progresses through a non-reversible lifecycle:

`queued → running → done` or `failed`

* **queued:** The job is accepted and waiting in the processing queue.
* **running:** The model is actively performing inference.
* **done:** The job completed successfully and predictions are available.
* **failed:** The job encountered an error (e.g., invalid image, timeout, internal failure).

**Example (in-progress response):**
```json
{
  "job_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "status": "running"
}