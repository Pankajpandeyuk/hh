# Storage Package: The Relational Vault & ACID Persistence

The **storage package** is the persistence backbone of the financial engine. While the core package handles validation math and business logic, the storage package manages direct interactions with PostgreSQL.

Its primary mission is to enforce strict ACID durability, execute pessimistic row-level locking to eliminate race conditions, and guarantee absolute immutability for all financial journal entries.

---

## Visual Structure & Data Flow

Within `com.fintech.ledger.storage`:

```plaintext
storage
│
├── accounts/      ──► [1. Account Vault]       ──► Pessimistic locking (SELECT FOR UPDATE) & balance persistence
├── transactions/  ──► [2. Transaction Ledger]  ──► Master transfer records, metadata & status management
└── entries/       ──► [3. Journal Entries]     ──► Immutable, append-only debit/credit line items
```

---

## Subfolders & Their Responsibilities

### 1. `accounts` (The Account & Locking Vault)
* **Purpose:** Manages account data and safeguards balances from concurrent modifications using database-level locking.
* **Estimated File Count:** 2 files.
* **Typical Files & Responsibilities:**
    * `AccountRepository.java`: Executes database queries that acquire row locks (`SELECT ... FOR UPDATE`) in deterministic, sorted order to prevent deadlocks.
    * `AccountPersistenceAdapter.java`: Translates pure domain account models into relational database entities.

### 2. `transactions` (The Transaction Metadata Manager)
* **Purpose:** Persists and manages the high-level metadata, tracking IDs, and lifecycle states of every financial transfer passing through the system.
* **Estimated File Count:** 2 files.
* **Typical Files & Responsibilities:**
    * `TransactionRepository.java`: Handles database CRUD operations for master transaction headers and records.
    * `TransactionEntity.java`: JPA entity mapping for transaction metadata (timestamps, idempotency reference keys, and final statuses).

### 3. `entries` (The Append-Only Journal Persistence)
* **Purpose:** Stores permanent, unalterable financial journal entries. In strict accounting, historical ledger rows are never updated or deleted; this subfolder ensures entries are strictly append-only.
* **Estimated File Count:** 2 files.
* **Typical Files & Responsibilities:**
    * `EntryRepository.java`: Handles the batch insertion of permanent debit and credit ledger rows.
    * `TransactionEntryEntity.java`: Relational mapping for individual journal lines, linking specific amounts and directions to accounts and parent transactions.

---

## How the 3 Subfolders Work Together in a Database Transaction

1. **Locking Accounts (`accounts`)**: Inside an atomic database transaction (`@Transactional`), `accounts` uses `SELECT ... FOR UPDATE` on sorted account IDs, safely blocking any parallel transactions trying to touch the same balances concurrently.
2. **Recording the Transfer (`transactions`)**: The `transactions` component persists the master transaction record to track that a transfer event has officially initiated.
3. **Writing Immutable Journal Lines (`entries`)**: The `entries` component appends the permanent, balance-altering debit and credit lines to the database table.

Once all three operations complete and commit successfully to PostgreSQL, the financial state update becomes permanent.
