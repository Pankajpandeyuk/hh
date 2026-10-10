# Architecture & System Design (`ARCHITECTURE.md`)

This document defines the high-level system architecture, package boundaries, fault isolation guarantees, and a **debugging & error-tracing matrix** for the **Core Financial Settlement & Double-Entry Ledger Engine**.

Designed around strict separation of concerns, the codebase is segregated into **six primary functional packages** under `com.fintech.ledger` to ensure that ingress, concurrency, business logic, domain rules, persistence, and event streaming are completely decoupled.

---

## System Bootstrap: `LedgerApplication.java`

`LedgerApplication.java` serves as the core entry point and runtime orchestrator:

* **Virtual Thread Execution**: Bootstraps the embedded web server configured to leverage **Java 21 Virtual Threads**, enabling thousands of concurrent I/O-bound financial requests with minimal memory overhead.
* **Infrastructure Initialization**: Establishes connection pools for PostgreSQL, initializes Redis cache/lock clients, and boots up background scheduled tasks for outbox event polling.

---

## High-Level Package Boundaries & Responsibilities

```text
com.fintech.ledger
│
├── LedgerApplication.java       # System bootstrap & virtual thread executor
│
├── gateway                      # 1. Ingress API Layer & Protocol Translation
├── deduplication                # 2. Distributed Concurrency & Idempotency Shield
├── core                         # 3. Orchestration Engine & Invariant Mathematics
├── ledger                       # 4. Pure Financial Domain Models & Rulebook
├── storage                      # 5. Relational Persistence & ACID Vault
└── streaming                    # 6. Transactional Outbox & Event Relay
```

### 1. `gateway` (The Ingress Boundary)

* **Architectural Role**: Serves as the outer perimeter facing clients, webhooks, and external microservices.
* **Core Duties**: Exposes versioned REST endpoints, binds incoming JSON payloads to immutable DTO contracts, validates basic syntax/constraints, and intercepts domain errors to return uniform, standardized HTTP error responses.

### 2. `deduplication` (The Concurrency Shield)

* **Architectural Role**: Acts as a distributed traffic bouncer and safety net powered by Redis.
* **Core Duties**: Prevents duplicate debits, race conditions, and network retry loops by extracting `Idempotency-Key` headers, fingerprinting request payloads with SHA-256 hashes, acquiring atomic distributed locks (`SETNX`), and serving cached response receipts.

### 3. `core` (The Accounting Brain & Orchestrator)

* **Architectural Role**: Coordinates the transactional pipeline and enforces accounting safety rules.
* **Core Duties**: Validates mathematical invariants (\(\sum \text{Debits} = \sum \text{Credits}\)), evaluates account fund sufficiency, and sorts account identifiers lexicographically to deterministically eliminate database deadlocks.

### 4. `ledger` (The Domain Rulebook)

* **Architectural Role**: Houses the pure, framework-agnostic business domain model.
* **Core Duties**: Defines immutable core entities (accounts, transactions, journal entry lines), type-safe financial classifications (`DEBIT`/`CREDIT`, account types, statuses), and domain-specific accounting exceptions.

### 5. `storage` (The Relational Persistence Vault)

* **Architectural Role**: Manages direct database interactions with PostgreSQL.
* **Core Duties**: Enforces strict ACID durability, executes pessimistic row-level locking (`SELECT ... FOR UPDATE`) in deterministic order, persists append-only journal entries (historical ledger rows are immutable), and commits atomic state changes.

### 6. `streaming` (The Asynchronous Event Relay)

* **Architectural Role**: Resolves the Dual-Write Problem using the Transactional Outbox Pattern.
* **Core Duties**: Atomically stages outgoing notification payloads into an outbox table within the exact same database transaction as the money transfer, running background poller workers to stream events reliably to Apache Kafka.

---

## Debugging & Error-Tracing Matrix (Finding Errors by Package)

When investigating bugs, performance bottlenecks, or production alerts, use this package-mapped troubleshooting guide to isolate the root cause instantly:

| Symptom / Error Type | Primary Suspect Package | Typical Root Cause / Action |
| --- | --- | --- |
| **HTTP `400 Bad Request` / Validation Failures** | `gateway` | Malformed JSON payloads, negative monetary amounts, missing required headers, or invalid currency codes. Check DTO validation constraints. |
| **HTTP `409 Conflict` / Concurrent Request Blocking** | `deduplication` | An identical `Idempotency-Key` is currently executing or payload fingerprint mismatch detected. Check Redis lock TTL or client retry behavior. |
| **Zero-Sum Math Violation (`ZeroSumViolationException`)** | `core` | The incoming transaction legs do not balance out (\(\sum \text{Debits} \neq \sum \text{Credits}\)). Inspect transaction splitting logic. |
| **Insufficient Funds Error (`InsufficientFundsException`)** | `core` & `storage` | Sender account balance is lower than the requested debit amount. Check available balance and concurrency locks. |
| **Database Deadlock / Thread Starvation** | `storage` & `core` | Accounts were locked in inconsistent orders. Verify that `core` is correctly sorting account IDs lexicographically before database locking. |
| **Events Not Reaching Kafka / Outbox Stalled** | `streaming` | Background poller worker failure, database connection drop, or Kafka broker downtime. Inspect outbox table status flags (`PENDING` vs `FAILED`). |

---

## End-to-End System Execution Flow

```text
[ Client / Webhook ]
          │
          ▼
    1. gateway          ──► Validates DTO schema & extracts Idempotency-Key
          │
          ▼
  2. deduplication      ──► Checks Redis lock & idempotency cache (returns 409 Conflict if busy)
          │
          ▼
       3. core          ──► Enforces Zero-Sum math & sorts account IDs to prevent deadlocks
          │
          ▼
      4. ledger         ──► Supplies domain entities, account types, and accounting rules
          │
          ▼
      5. storage        ──► Atomic DB Transaction: locks rows (SELECT FOR UPDATE), updates balances,
          │                  writes immutable journal entries & stages outbox record
          │
          ▼
     6. streaming       ──► Background worker polls outbox table & streams events to Apache Kafka
          │
          ▼
    [ gateway ]         ──► Returns HTTP 200/201 Success Receipt to Client
```
