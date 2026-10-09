# Gateway Package: The Ingress API Boundary & Protocol Translation

The **gateway package** serves as the external face of the financial engine. It acts as the primary ingress boundary that receives HTTP requests from web clients, mobile apps, or payment webhook providers.

Its core responsibility is to translate external network protocols into clean, validated domain data contracts, while ensuring uniform error responses back to the client.

---

## Visual Structure & Data Flow

Within `com.fintech.ledger.gateway`:

```plaintext
gateway/
├── routes/
│   ├── TransferController.java        # REST endpoints for executing financial transfers and lookups
│   └── AccountController.java         # REST endpoints for account creation, balance checks, and status queries
├── schemas/
│   ├── TransferRequestDto.java        # Incoming transfer payload validation contract (amounts, currency, keys)
│   ├── TransferResponseDto.java       # Standardized success receipt returned to clients
│   ├── AccountCreateRequestDto.java   # Onboarding request payload for new accounts
│   └── AccountResponseDto.java        # Account details and current balance representation
└── errors/
    ├── GlobalExceptionHandler.java    # Centralized @ControllerAdvice mapping domain errors to HTTP statuses
    └── ApiErrorResponse.java          # Uniform error payload structure (timestamps, codes, messages)
```

---

## Subfolders & Responsibilities

### 1. `routes` (The API Endpoints)
* **Purpose:** Exposes clean, versioned RESTful endpoints that external clients and upstream services can call.
* **Estimated File Count:** 2 to 3 files.
* **Typical Files & Responsibilities:**
    * `TransferController.java`: Exposes REST endpoints for initiating financial transfers, querying transaction receipts, and checking settlement status.
    * `AccountController.java`: Manages HTTP routes for creating accounts, retrieving balances, and inspecting ledger registries.

### 2. `schemas` (The Data Contracts & DTOs)
* **Purpose:** Strictly decouples internal database models from external API structures. It uses validation annotations to ensure incoming data is clean before processing begins.
* **Estimated File Count:** 3 to 4 files.
* **Typical Files & Responsibilities:**
    * `TransferRequestDto.java`: Captures incoming transfer payloads (sender ID, receiver ID, amount, currency) and enforces constraints like positive monetary values and non-null idempotency headers.
    * `TransferResponseDto.java`: Defines the clean JSON receipt format returned to the client upon successful execution.
    * `AccountCreateRequestDto.java`: Validates input payloads for onboarding new accounts into the ledger system.

### 3. `errors` (The Global Exception & Error Handler)
* **Purpose:** Intercepts low-level domain exceptions (such as `InsufficientFundsException` or `ZeroSumViolationException`) and cleanly translates them into standardized, user-friendly HTTP error codes and JSON messages.
* **Estimated File Count:** 2 files.
* **Typical Files & Responsibilities:**
    * `GlobalExceptionHandler.java`: A centralized Spring `@ControllerAdvice` that catches runtime exceptions across the application and maps them to appropriate HTTP statuses (e.g., `400 Bad Request`, `409 Conflict`, `422 Unprocessable Entity`).
    * `ApiErrorResponse.java`: Defines a uniform error payload structure (containing error codes, descriptive messages, timestamps, and tracking paths) for API consumers.

---

## Dynamic Data Flow Summary

1. **Ingress & Binding (`routes` & `schemas`)**: An HTTP POST request hits `TransferController` (`routes`). Spring Boot automatically parses the incoming JSON body and maps it to `TransferRequestDto` (`schemas`), validating constraints like positive amounts.
2. **Execution & Interception (`errors`)**: The controller hands the validated request down to the deduplication and core layers. If a business rule fails (e.g., the account has insufficient funds), a domain exception is thrown. `GlobalExceptionHandler` (`errors`) instantly catches this exception and returns a clean, standardized HTTP error response to the client without exposing raw stack traces.
3. **Success Response**: If execution succeeds, the resulting data is wrapped into a success schema and returned via the router to the client.
