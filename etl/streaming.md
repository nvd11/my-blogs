# Streaming Data Engineering & Architecture: GCP Pub/Sub & Cloud Dataflow

This document serves as an architectural blueprint and a high-frequency interview preparation guide for building and operating real-time, event-driven, and highly resilient compliance streaming pipelines. It is designed to match the SRE, cost-optimization, and risk governance standards of a tier-1 investment bank.

---

## 1. RCDP Real-Time Streaming Architecture (The Blueprint)

In the Regulatory Compliance Data Platform (RCDP), our streaming architecture is built on the principle of **event-driven, loose coupling**.

```
  [ Upstream App / Rapid2 ] (Continuous Ingestion)
               │
               ▼ (Avro/JSON Payload)
    [ GCP Pub/Sub Ingestion ] (Message Ingestion & At-Least-Once Delivery)
               │
               ▼ (Streaming Fetch / Pull)
  [ GCP Cloud Dataflow (Apache Beam) ] (Streaming ETL / Pipelining)
        │                      │
        │ (Healthy Streams)    │ (Malformed/Corrupted Payload - Side Output)
        ▼                      ▼
  [ BigQuery DW ]       [ GCS / Pub/Sub DLQ ] (Dead Letter Queue for Audit)
```

### 1.1 Ingestion Layer: GCP Pub/Sub
*   **Role**: Serves as our globally distributed, highly available message bus.
*   **Design**: Ingests real-time transactional and regulatory alerts from upstream systems (e.g., Rapid2). Upstream triggers are pushed into dedicated Pub/Sub topics.

### 1.2 Computation Layer: GCP Cloud Dataflow (Apache Beam)
*   **Role**: A serverless, auto-scaling execution engine running Apache Beam pipelines.
*   **Processing**: Performs schema validation, enrichment (joining streaming records with slow-moving lookup tables like Helios/Regmap), deduplication, and routing.

### 1.3 Sink/Storage Layer: BigQuery
*   **Role**: Stores structured, near real-time data for analytical queries, model training (CMLP), and downstream API serving (via Authorized Views).

---

## 2. Deep Dive: Streaming Engineering Bottlenecks & Countermeasures

### 2.1 Guaranteeing Exactly-Once Processing
GCP Pub/Sub natively guarantees **At-Least-Once** delivery. Duplicate messages can occur due to network retries, connection timeouts, or subscriber restarts. To guarantee **Exactly-Once** semantics end-to-end (from Pub/Sub to BigQuery):

1.  **Dataflow Deduplication**: Dataflow natively integrates with Pub/Sub and tracks message IDs over a sliding window (typically 10 minutes) to drop duplicates.
2.  **Idempotent Sinks (BigQuery)**: If writing to BigQuery, we assign a unique, deterministic `insertId` (e.g., hashing the business primary key + event timestamp) for each row. BigQuery uses this `insertId` to automatically deduplicate records streamed within a 1-minute window.
3.  **Application-Level Deduplication**: For stateful operations, we use Apache Beam's `State and Timer API` to store seen transaction IDs in a localized state and discard duplicates before downstream processing.

### 2.2 Flow Control & Backpressure Management
Under extreme market volatility or batch alert replay, message volume can spike exponentially, risking downstream service crashes or BigQuery quota exhaustion.

*   **Pub/Sub Flow Control**: We configure subscription-level `flow_control_settings` (limiting outstanding messages and bytes) in our Dataflow runner to prevent overloading Worker VM memory.
*   **Dataflow Autoscaling**: Dataflow automatically scales workers up or down based on **Backlog (System Lag)** and CPU utilization. We set `max_num_workers` to capped values to prevent runaway costs, and enable **Streaming Engine** to offload backpressure tracking from Worker VMs to Google’s managed control plane.

### 2.3 Resilient Error Handling: The Dead Letter Queue (DLQ)
A single corrupt or malformed payload must **never** block or crash a streaming pipeline. We enforce a zero-loss policy for auditing purposes:

*   **Side Outputs (TupleTags)**: We wrap our JSON/Avro parsing logic inside a `try-catch` block. 
*   **Routing**: Healthy records are emitted through the main output stream. Corrupted, invalid, or schema-violating records are routed to a **Dead Letter Queue (DLQ)** using Apache Beam's `Side Outputs`.
*   **Auditability**: The DLQ serializes the raw corrupt string alongside the exact Java/Python stack trace and event timestamp, dumping it into a GCS bucket or a dedicated Pub/Sub topic for alerting and manual replay.

### 2.4 Late-Arriving Data, Watermarks, and Windows
In distributed banking environments, network delays can cause data to arrive out of order (e.g., a trade from 14:00 PM arriving at 14:15 PM).

*   **Event Time vs. Processing Time**: We partition and window our streaming data exclusively based on **Event Time** (the timestamp of the trade/alert occurrence) rather than Processing Time (when Dataflow receives it).
*   **Heuristic Watermarks**: Dataflow calculates a **Watermark**—a moving temporal boundary representing its progress. It estimates that no more data prior to timestamp $T$ will arrive.
*   **Allowed Lateness**: We configure `withAllowedLateness(Duration.standardMinutes(15))` to handle delayed events. Late data arriving after the watermark but before the lateness threshold triggers an updated, accumulative firing of the window to BigQuery. Data arriving past the threshold is discarded or routed to the DLQ.

---

## 3. High-Frequency Interview Q&A (The Director's Corner)

### Q1: How do you handle Data Skewness (Hot Keys) in a streaming Join or GroupByKey?
> **Answer**: 
> "Data skewness occurs when a disproportionately large volume of streaming events share the same key (e.g., a dominant corporate client or a default null value), leading to a single Dataflow Worker VM bottlenecking on CPU/Memory while others sit idle.
> 
> My strategy to resolve this involves:
> 1. **Filtering Null Keys Early**: If the skewness is caused by unmapped or null IDs (e.g., uncompleted transaction details), we filter or route those out before the `GroupByKey` stage.
> 2. **Salting the Keys (Key Sharding)**: We dynamically append a random integer suffix (a 'salt' between $0$ and $N-1$) to the hot key (e.g., `Client_A` becomes `Client_A_3`). We perform the localized aggregation on the salted keys first, and then run a second global aggregation after stripping the salt.
> 3. **Using Beam Combiners**: Instead of a full `GroupByKey` which forces all values to gather at a single node, we leverage pre-aggregation via `Combine.perKey()`. This runs map-side combiner reduction within individual worker JVMs, massively reducing network shuffle traffic."

### Q2: Streaming pipelines are notoriously expensive. How do you optimize Dataflow costs in production?
> **Answer**:
> "Cost optimization in production-grade streaming is about balancing SLA latency requirements against resource consumption. We tackle this through several architecture-level decisions:
> 
> 1. **Enable Streaming Engine**: We shift the execution of shuffle and state management from the Dataflow Worker VMs to a dedicated service backend. This reduces Worker CPU/memory requirements, allowing us to use cheaper, smaller VM instances.
> 2. **Tune Machine Types and Autoscaling**: We don't blindly accept default VM types. We profile memory/CPU utilization and often choose high-memory/low-CPU instances if our transforms are non-compute heavy. We also hard-cap `max_num_workers` to prevent runaway autoscaling costs.
> 3. **Optimize BigQuery Writes**: We prefer **BigQuery Storage Write API** over legacy streaming inserts, as the Storage Write API offers dramatically lower costs per GB ingested and superior throughput.
> 4. **Micro-batching Evaluation**: We constantly challenge the business requirement. If the compliance officers only review alerts once an hour, we don't run a 24/7 active streaming pipeline. Instead, we configure a cron-scheduled micro-batch Dataflow job that fires every hour, reducing idle compute waste."

### 3. How do you guarantee "Exactly-once" end-to-end delivery when interacting with third-party APIs or relational databases?
> **Answer**:
> "While GCP Pub/Sub to BigQuery guarantees Exactly-Once natively, interacting with external systems (like calling a third-party risk analysis API or writing to an on-prem relational DB) breaks this guarantee because those operations represent **side-effects** that cannot be automatically rolled back by the Dataflow runner.
> 
> To guarantee Exactly-Once in these scenarios, we must design for **Idempotency**:
> 1. **Idempotent API Contracts**: We collaborate with external teams to ensure their APIs accept an idempotency key (e.g., a UUID generated from hashing the unique transaction metadata). If Dataflow retries a failed bundle and calls the API twice with the same key, the external system simply returns the cached response rather than executing a duplicate trade or alert.
> 2. **Upsert Operations on Sinks**: When writing streaming outputs to external RDBMS, we avoid plain `INSERT` statements. We implement `UPSERT` (using `ON CONFLICT DO UPDATE` or merge operations) keyed on a deterministic primary ID, ensuring that retried bundles safely overwrite existing records rather than duplicating them."

### Q4: How do you monitor the operational health of your streaming pipelines? What KPIs do you track?
> **Answer**:
> "Monitoring streaming pipelines is fundamentally different from batch monitoring. We don't look at success/failure on exit; we track **flow, latency, and resource constraints**.
> 
> Our critical metrics in Cloud Monitoring (Stackdriver) and our Grafana Dashboards include:
> 1. **System Lag (Backlog)**: The time difference between when the oldest unacknowledged message was published and the current time. A spiking System Lag is our primary alert trigger, indicating that the pipeline is falling behind and needs to scale up.
> 2. **Dataflow Watermark Delay**: The gap between the current time and the watermark. A wide gap indicates late-arriving data or processing bottlenecks.
> 3. **Dataflow CPU and Memory Utilization**: To ensure our workers are sized correctly.
> 4. **Throttling & Backpressure**: We monitor if the pipeline is actively being throttled by upstream limits or downstream database connection pools.
> 5. **DLQ Message Rate**: A sudden spike in the Dead Letter Queue error rate immediately triggers a P1 alert, signaling a potential upstream schema breakage or a corrupted release."
