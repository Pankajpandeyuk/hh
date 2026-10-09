# Deduplication Package: The Distributed Concurrency Shield & Idempotency Gateway

The **`deduplication`** package acts as the system's traffic bouncer and safety net. In high-volume financial platforms, unstable client connections often trigger automatic retries, double-clicks, or parallel network requests. This package leverages **Redis** to ensure that every unique transaction request is processed exactly once, preventing duplicate debits and race conditions before they ever touch the database.

---

## Visual Structure & Data Flow Within `deduplication`

```text
deduplication/
├── keys/
│   ├── IdempotencyKeyExtractor.java   # Extracts and validates the Idempotency-Key HTTP header
│   └── PayloadHasher.java             # Generates cryptographic SHA-256 hashes of request bodies
├── locks/
│   ├── RedisLockManager.java          # Manages atomic Redis distributed locks (SETNX with TTL)
│   └── LockAcquisitionException.java  # Exception thrown when concurrent duplicate requests collide (409 Conflict)
└── cache/
    ├── ReceiptCacheRepository.java    # Saves and retrieves completed transaction receipts in Redis
    └── ReplayHandler.java             # Intercepts repeated keys to replay cached receipts instantly
```

---

## The 3 Subfolders & Their Responsibilities

### 1. `keys` (The Fingerprint Extractor)

* **Why it's here**: To identify who is making the request and ensure the payload hasn't been maliciously or accidentally altered while using the same key.
* **Estimated File Count**: 2 files.

<details>
<summary>View Typical Files & Responsibilities</summary>

* **`IdempotencyKeyExtractor.java`**: Extracts the `Idempotency-Key` header from incoming HTTP requests.
* **`PayloadHasher.java`**: Generates a cryptographic SHA-256 hash of the request body. If a client reuses an idempotency key with a different payload, this component detects the mismatch and rejects it instantly.
</details>

---

### 2. `locks` (The Distributed Concurrency Guard)

* **Why it's here**: To prevent simultaneous execution when two identical requests hit different server instances at the exact same millisecond.
* **Estimated File Count**: 2 files.

<details>
<summary>View Typical Files & Responsibilities</summary>

* **`RedisLockManager.java`**: Uses Redis atomic primitives (`SETNX` with a short time-to-live) to safely acquire and release distributed locks.
* **`LockAcquisitionException.java / Interceptor`**: If a request comes in while an identical transaction is currently running, this component immediately intercepts it and returns an HTTP 409 Conflict status.
</details>

---

### 3. `cache` (The Receipt Replay Vault)

* **Why it's here**: To instantly serve responses for transactions that have already successfully finished, eliminating redundant database queries during network retries.
* **Estimated File Count**: 2 files.

<details>
<summary>View Typical Files & Responsibilities</summary>

* **`ReceiptCacheRepository.java`**: Saves the final transaction receipt (HTTP status, transfer ID, timestamp) into Redis with a defined TTL (Time-To-Live).
* **`ReplayHandler.java`**: Intercepts completed idempotency keys and streams back the cached JSON receipt instantly without hitting the core database.
</details>

---

## How the 3 Subfolders Work Together

1. **Extraction (`keys`)**: The incoming request hits the gateway, and `keys` extracts the `Idempotency-Key` and computes the SHA-256 payload fingerprint.
2. **Locking & Guarding (`locks`)**: `locks` attempts to acquire an atomic lock in Redis using that key.
    * *If a lock already exists because the transaction is currently processing, it blocks and returns an HTTP 409 Conflict.*
3. **Replay Check (`cache`)**: `cache` checks if a previous execution already finished.
    * *If a cached receipt exists, it bypasses the entire database and returns the original success response instantly.*
    * *If the key is completely new, the lock is secured, and execution flows down to `core` and `storage`. Once finished, the final receipt is saved back into the `cache`.*
