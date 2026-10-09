# Ledger Package: The Financial Domain Core & Accounting Rules

The **ledger package** is the pure business domain layer of the application. While other packages manage HTTP protocols (`gateway`), concurrency caching (`deduplication`), or database mechanics (`storage`), the ledger package houses the platonic ideals of double-entry bookkeeping.

It is completely decoupled from frameworks, containing pure business entities, strict financial classifications, and domain-specific exceptions.

---

## Visual Structure & Data Flow

Within `com.fintech.ledger.ledger`:

```plaintext
ledger
│
├── models/        ──► [1. Domain Entities]    ──► Immutable data representations (Account, Transaction, Entry)
├── types/         ──► [2. Financial Enums]    ──► Strict classifications (DEBIT/CREDIT, AccountType, Status)
└── exceptions/    ──► [3. Domain Errors]      ──► Accounting rule infractions (InsufficientFunds, ZeroSum)
```

---

## Subfolders & Their Responsibilities

### 1. `models` (The Domain Entities & Value Objects)
* **Purpose:** Defines the fundamental data structures that represent real-world financial artifacts (money, ledgers, accounts, and journal lines) following strict double-entry accounting rules.
* **Estimated File Count:** 4 files.
* **Typical Files & Responsibilities:**
    * `Ledger.java`: Represents the overarching financial ledger boundary or book of accounts.
    * `Account.java`: Represents an individual account entity (e.g., user wallet, platform fee account, merchant reserve) holding balance counters.
    * `LedgerTransaction.java`: Represents a balanced financial transfer event containing metadata, timestamps, and idempotency tracking references.
    * `TransactionEntry.java`: Represents an individual immutable debit or credit ledger line (journal entry) tied to a transaction.

### 2. `types` (The Financial Classifications & Enums)
* **Purpose:** Enforces type safety across the application, preventing arbitrary strings from being used for financial directions, account states, or ledger categories.
* **Estimated File Count:** 3 files.
* **Typical Files & Responsibilities:**
    * `EntryDirection.java`: Defines the core double-entry accounting directions: `DEBIT` and `CREDIT`.
    * `AccountType.java`: Categorizes accounts based on accounting principles (e.g., `ASSET`, `LIABILITY`, `EQUITY`, `REVENUE`, `EXPENSE`).
    * `TransactionStatus.java`: Tracks the lifecycle state of a transaction (e.g., `PENDING`, `COMMITTED`, `REJECTED`, `REVERSED`).

### 3. `exceptions` (The Domain Accounting Errors)
* **Purpose:** Encapsulates specific financial rule violations as distinct, expressive Java exceptions rather than generic runtime errors.
* **Estimated File Count:** 2 to 3 files.
* **Typical Files & Responsibilities:**
    * `InsufficientFundsException.java`: Thrown when a debit attempt exceeds the available balance of an account.
    * `ZeroSumViolationException.java`: Thrown when transaction legs fail the mathematical balance check (\(\sum \text{Debits} \neq \sum \text{Credits}\)).
    * `AccountFrozenException.java`: Thrown when an attempted transaction involves an account that is suspended, closed, or frozen.

---

## How the 3 Subfolders Work Together

1. **Classification (`types`)**: When a transfer is structured, `types` provides the strict constants (`DEBIT` and `CREDIT`) required to label transaction lines correctly.
2. **Instantiation (`models`)**: The core and storage packages use these types to build valid domain objects—instantiating an `Account` and wrapping multiple `TransactionEntry` lines inside a unified `LedgerTransaction`.
3. **Guardrails (`exceptions`)**: If business invariants are broken during this process (such as a negative balance or an unbalanced debit-credit sum), the domain layer throws precise errors from `exceptions`. These are ultimately caught upstream by the gateway's global exception handler to be mapped to clean HTTP responses.
