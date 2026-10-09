# Core Package: The Accounting Brain & Execution Engine

The **`core`** package is the central nervous system of the financial ledger. While the `gateway` handles ingress protocols and `storage` handles database persistence, the `core` package houses all the **business logic, mathematical invariants, race-condition defenses, and transaction orchestration**.

---

## Visual Structure & Data Flow Within `core`

```text
core/
├── processor/
│   ├── TransactionProcessor.java      # Master coordinator executing the multi-step transfer pipeline
│   ├── TransferOrchestrator.java      # Routes multi-leg splits (merchant payables, platform fees, taxes)
│   └── DeadlockPreventionService.java # Sorts account IDs lexicographically before lock acquisition
├── validation/
│   ├── ZeroSumValidator.java          # Enforces the mathematical invariant: Sum of Debits == Sum of Credits
│   └── TransactionStructureValidator.java # Validates line item directions, currencies, and positive values
└── balance/
    ├── BalanceChecker.java            # Evaluates sender fund sufficiency against available balances
    └── AccountStateEvaluator.java     # Inspects account status flags (ACTIVE, FROZEN, CLOSED)
```

---

## The 3 Subfolders & Their Responsibilities

### 1. `processor` (The Master Coordinator)

* **Why it's here**: A financial transfer is a multi-step workflow (sorting accounts, locking rows, checking balances, writing entries, and publishing outbox events). The `processor` package contains the master services that coordinate this entire pipeline from start to finish.
* **Estimated File Count**: 2 to 3 files.

<details>
<summary>View Typical Files & Responsibilities</summary>

* **`TransactionProcessor.java`**: The core service bean that executes the main transfer pipeline within a transactional boundary.
* **`TransferOrchestrator.java`**: Handles multi-leg transaction routing, splitting money across accounts (e.g., merchant payable, platform fee, tax).
* **`DeadlockPreventionService.java`**: Automatically sorts all account IDs lexicographically before any lock is requested to prevent database deadlocks.
</details>

---

### 2. `validation` (The Accounting Invariant Enforcer)

* **Why it's here**: To guarantee that money is never magically created or destroyed. This subfolder strictly enforces fundamental double-entry accounting rules before any write operation is permitted to touch the database.
* **Estimated File Count**: 2 files.

<details>
<summary>View Typical Files & Responsibilities</summary>

* **`ZeroSumValidator.java`**: Verifies the foundational invariant formula where total debits must equal total credits (\(\sum \text{Debits} - \sum \text{Credits} = 0\)). If this check fails, the transaction is rejected instantly.
* **`TransactionStructureValidator.java`**: Validates that incoming transaction lines contain valid directions, non-null currencies, and positive monetary amounts.
</details>

---

### 3. `balance` (The Fund Sufficiency & Limit Guard)

* **Why it's here**: To prevent negative balances, unauthorized overdrafts, and race conditions where two simultaneous requests try to spend the same funds. It works closely with the storage layer's row-level locks.
* **Estimated File Count**: 2 files.

<details>
<summary>View Typical Files & Responsibilities</summary>

* **`BalanceChecker.java`**: Evaluates whether the sender account has sufficient available balance to complete a debit transaction.
* **`AccountStateEvaluator.java`**: Inspects account status flags (e.g., ACTIVE, FROZEN, CLOSED) to ensure transactions can only occur on valid accounts.
</details>

---

## How the 3 Subfolders Work Together in a Transaction

1. **Ingress Phase**: The request enters the `processor` (`TransactionProcessor`).
2. **Invariant Phase**: The processor hands the transaction legs to `validation` (`ZeroSumValidator`) to ensure the math balances out (Debits == Credits).
3. **Sufficiency Phase**: The processor calls `balance` (`BalanceChecker`) to confirm the sender's account has enough funds.
4. **Execution Phase**: Once validated and checked, the processor securely sorts the account IDs to avoid deadlocks and passes execution to the `storage` layer for atomic persistence.
