# Core Financial Settlement & Double-Entry Ledger Engine

[![Java 21](https://img.shields.io/badge/Java-21-orange.svg)](https://www.oracle.com/java/)
[![Spring Boot 3.x](https://img.shields.io/badge/Spring%20Boot-3.x-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-ACID-blue.svg)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-Distributed%20Lock-red.svg)](https://redis.io/)
[![Apache Kafka](https://img.shields.io/badge/Apache%20Kafka-Outbox%20Streaming-orange.svg)](https://kafka.apache.org/)
[![Apache Maven](https://img.shields.io/badge/Apache%20Maven-Build-C71A36.svg)](https://maven.apache.org/)
[![Spring Boot Maven Plugin](https://img.shields.io/badge/Spring%20Boot-Maven%20Plugin-brightgreen.svg)](https://docs.spring.io/spring-boot/4.1.1/maven-plugin)
[![OCI Image](https://img.shields.io/badge/OCI-Image%20Build-blue.svg)](https://opencontainers.org/)
[![Spring Boot Testcontainers](https://img.shields.io/badge/Spring%20Boot-Testcontainers-brightgreen.svg)](https://docs.spring.io/spring-boot/4.1.1/reference/testing/testcontainers.html#testing.testcontainers)
[![Testcontainers PostgreSQL](https://img.shields.io/badge/Testcontainers-PostgreSQL-4169E1.svg)](https://java.testcontainers.org/modules/databases/postgres/)
[![Spring Web](https://img.shields.io/badge/Spring%20Web-REST%20API-brightgreen.svg)](https://spring.io/projects/spring-framework)
[![Spring Data JPA](https://img.shields.io/badge/Spring%20Data-JPA-brightgreen.svg)](https://spring.io/projects/spring-data-jpa)
[![Spring Data Redis](https://img.shields.io/badge/Spring%20Data-Redis-red.svg)](https://spring.io/projects/spring-data-redis)
[![Spring Validation](https://img.shields.io/badge/Spring-Validation-brightgreen.svg)](https://docs.spring.io/spring-boot/4.1.1/reference/io/validation.html)
[![Flyway Migration](https://img.shields.io/badge/Flyway-Database%20Migration-CC0200.svg)](https://www.red-gate.com/products/flyway/)
[![Testcontainers](https://img.shields.io/badge/Testcontainers-Integration%20Testing-2496ED.svg)](https://testcontainers.com/)


An enterprise-grade, distributed **Core Financial Settlement & Double-Entry Ledger Engine** modeled after modern banking rails and payment platforms like Stripe and Adyen. Designed from the ground up for zero-drift financial accuracy, high concurrency, strict idempotency, and reliable event streaming.

---

## Key Architectural Invariants

* **Strict Immutability (Append-Only):** Ledger entry rows are never updated or deleted. Corrections are handled exclusively via compensating reversal transactions.
* **Zero-Sum Balance Rule:** Every multi-leg transaction must satisfy \(\sum \text{Debits} - \sum \text{Credits} = 0\) before touching account balances.
* **Pessimistic Row-Level Locking:** Accounts are protected using database-level row locks (`SELECT ... FOR UPDATE`) sorted lexicographically to eliminate deadlocks.
* **Distributed Idempotency:** Guarded by Redis (`SETNX` and cryptographic request fingerprinting) to instantly return cached receipts or reject concurrent duplicate requests with an HTTP `409 Conflict`.
* **Transactional Outbox Pattern:** Database updates and outgoing message payloads are committed within the exact same atomic transaction, ensuring zero data loss before asynchronous relay to Apache Kafka.

---

## Technology Stack & Dependencies

The project is built on a high-performance modern tech stack optimized for massive throughput and thread efficiency:

| Component                        | Technology | Purpose |
|:---------------------------------| :--- | :--- |
| **Runtime & Language**           | Java 21 | Leverages **Virtual Threads** (Project Loom) to handle thousands of concurrent I/O-bound requests with minimal memory overhead. |
| **Framework**                    | Spring Boot 3.x | Core application container managing REST controllers, dependency injection, and declarative transactions (`@Transactional`). |
| **Relational Vault**             | PostgreSQL | Single source of truth providing strict ACID guarantees, constraints, and pessimistic locking. |
| **Concurrency Shield**           | Redis | In-memory key-value store for atomic distributed locks, request deduplication, and TTL response caching. |
| **Event Streaming**              | Apache Kafka | Distributed message broker receiving completed settlement events from the outbox relay. |
| **Container Runtime**            | Docker & Compose | Local orchestration for isolated PostgreSQL, Redis, and Kafka infrastructure. |
| **Build Automation**             | Apache Maven | Manages project dependencies, build lifecycle, and application packaging. |
| **Application**                  | Spring Boot Maven Plugin | Builds executable Spring Boot JARs and supports OCI image creation. |
| **Container Image Standard**     | OCI Image | Provides a standardized container image format for consistent deployment. |
| **REST API Layer**               | Spring Web | Handles REST controllers, HTTP requests, and API endpoint development. |
| **Persistence Layer**            | Spring Data JPA | Simplifies database access through repositories, ORM mapping, and transaction-aware persistence. |
| **Redis Integration**            | Spring Data Redis | Integrates Redis for distributed locking, atomic operations, request deduplication, and caching. |
| **Input Validation**             | Spring Validation | Validates incoming API requests and enforces input constraints before business processing. |
| **Database Migration**           | Flyway | Versions and applies database schema changes through controlled, versioned migrations. |
| **Integration Testing**          | Testcontainers | Runs disposable infrastructure containers for realistic integration tests. |
| **Database Integration Testing** | Testcontainers PostgreSQL | Provides isolated PostgreSQL instances to test queries, constraints, and transaction behavior against a real database. |
| **Spring Boot Testing**          | Spring Boot Testcontainers | Integrates Testcontainers with Spring Boot tests to simplify container lifecycle management and application integration testing. |

### Core Project Dependencies
* `spring-boot-starter-web` (REST API & Embedded Tomcat/Undertow)
* `spring-boot-starter-data-jpa` (Hibernate ORM & Database Persistence)
* `spring-boot-starter-data-redis` (Jedis/Lettuce Redis client integration)
* `spring-kafka` (Apache Kafka producer/consumer template support)
* `postgresql` (PostgreSQL JDBC Driver)
* `lombok` (Boilerplate reduction)
* `flyway-core` (Database schema migration versioning)
* `spring-boot-starter-validation` (Bean Validation for request DTOs and input constraints)
* `spring-boot-starter-test` (JUnit, Mockito & Spring Boot testing support)
* `spring-boot-testcontainers` (Spring Boot integration with Testcontainers)
* `org.testcontainers:postgresql` (PostgreSQL containers for integration testing)
* `org.testcontainers:junit-jupiter` (JUnit 5 lifecycle integration for Testcontainers)
* `spring-boot-maven-plugin` (Spring Boot application packaging & OCI image building)
* `maven-compiler-plugin` (Java compilation & annotation processor configuration)

---

## Repository Structure

```text
com.fintech.ledger
│
├── LedgerApplication.java       # Spring Boot main entry point & thread configuration
│
├── gateway                      # REST controllers, DTO data contracts, and global exception handlers
├── deduplication                # Redis distributed locks, payload hashing, and idempotency interceptors
├── core                         # Orchestration engine, zero-sum verification, and balance checks
├── ledger                       # Financial domain entities, enums (DEBIT/CREDIT), and custom exceptions
├── storage                      # PostgreSQL persistence, row-level locking queries, and append-only entries
└── streaming                    # Transactional outbox polling workers and Kafka event dispatchers
```

---

## Getting Started & Local Development

### Prerequisites
* **Java Development Kit (JDK) 21+** installed.
* **Docker & Docker Compose** for spinning up local infrastructure services.

### Execution Steps

#### 1. Clone the Repository
```bash
git clone https://github.com/pankajpandey22/core-financial-ledger.git
cd core-financial-ledger
```

#### 2. Spin Up Infrastructure Services
Start PostgreSQL, Redis, and Apache Kafka locally using Docker Compose:
```bash
docker-compose up -d
```

#### 3. Configure Application Properties
Verify your `src/main/resources/application.yml` or `application.properties` points to the local infrastructure endpoints:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ledger_db
    username: postgres
    password: password
  data:
    redis:
      host: localhost
      port: 6379
  kafka:
    bootstrap-servers: localhost:9092
```

#### 4. Build and Run the Application
Run the Spring Boot application using the Maven wrapper:
```bash
./mvnw spring-boot:run
```

---

## License
This project is licensed under the terms of the **MIT License**.
