# Streaming Package: The Asynchronous Event Relay & Outbox Pipeline

The **streaming package** serves as the asynchronous communication bridge of the financial engine. In a distributed architecture, updating internal bank accounts and notifying downstream services (such as fraud detection, analytics, audit logs, or push notifications) must never suffer from the Dual-Write Problem—where a database update succeeds but the network message to Kafka fails.

This package solves that problem by implementing the **Transactional Outbox Pattern**.

---

## Visual Structure & Data Flow

Within `com.fintech.ledger.streaming`:

```plaintext
streaming
│
├── outbox/        ──► [1. Outbox Staging]      ──► Atomic staging table for pending event records
├── publisher/     ──► [2. Background Poller]   ──► Scheduled relay worker tracking un-dispatched messages
└── dispatcher/    ──► [3. Kafka Producer]      ──► Delivers event payloads to Apache Kafka topics
```

---

## Subfolders & Their Responsibilities

### 1. `outbox` (The Event Staging Vault)
* **Purpose:** Guarantees that no system event is ever lost. By writing outgoing notification payloads into an outbox table inside the exact same database transaction as the money transfer, it ensures absolute atomicity.
* **Estimated File Count:** 2 files.
* **Typical Files & Responsibilities:**
    * `OutboxRepository.java`: Handles database queries to fetch pending, unpublished outbox records and mark them as processed.
    * `OutboxEventEntity.java`: JPA entity mapping for the outbox table, holding payload data, event types, creation timestamps, and publication status flags.

### 2. `publisher` (The Background Relay Worker)
* **Purpose:** Decouples synchronous HTTP client responses from slow or unstable external message brokers (like Kafka). It runs in the background, polling for work without blocking the user's transfer request.
* **Estimated File Count:** 2 files.
* **Typical Files & Responsibilities:**
    * `OutboxPoller.java`: A scheduled background worker (using Spring's `@Scheduled`) that continuously scans the outbox table for unprocessed records.
    * `OutboxRelayService.java`: Manages batching logic, error backoffs, and status transitions (moving records from `PENDING` to `PUBLISHED` or `FAILED`).

### 3. `dispatcher` (The Kafka Event Producer)
* **Purpose:** Transforms internal outbox records into standardized event streams and pushes them to Apache Kafka with at-least-once delivery guarantees.
* **Estimated File Count:** 2 files.
* **Typical Files & Responsibilities:**
    * `KafkaEventProducer.java`: Wraps Spring Kafka's `KafkaTemplate` to publish structured JSON messages to designated financial topics (e.g., `ledger.transactions.settled`).
    * `EventPayloadSerializer.java`: Serializes domain transfer receipts and journal logs into clean, versioned message schemas for downstream consumers.

---

## How the 3 Subfolders Work Together in the Event Lifecycle

1. **Staging Atomically (`outbox`)**: When a money transfer commits in PostgreSQL via the storage layer, an event entry is simultaneously written into the outbox table within that exact same database transaction.
2. **Polling for Work (`publisher`)**: The background `OutboxPoller` wakes up on a fixed schedule, queries the outbox table for pending records, and passes them to the relay service.
3. **Dispatching to Kafka (`dispatcher`)**: The relay service hands the records to the `KafkaEventProducer`, which streams them out to Apache Kafka.

Once Kafka acknowledges receipt, the outbox record is marked as successfully published, keeping the system fully synchronized.
