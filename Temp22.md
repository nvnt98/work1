# InfyBank360 - Requirement Specification

> **Note to the reader (Navneet):** This document describes **what** the InfyBank360 system must do and **why**, based on the newly defined architectural vision. It intentionally avoids implementation details (Java code, Spring annotations, database schemas), which will be covered later in the technical specification (`spec.md`). This fresh specification consolidates the domain into 7 core microservices, introduces event-driven communication for cross-cutting concerns, and utilizes a multi-database approach (Relational, Document Store, and Cache) to meet specific performance and flexibility needs.

---

## 1. Document Information

| Field | Value |
|---|---|
| **Project Name** | InfyBank360 |
| **Document Title** | Requirement Specification (Fresh Architecture) |
| **Document Version** | 2.0 |
| **Date** | October 07, 2026 |
| **Author** | Business Analysis & Architecture Team |
| **Status** | Draft — Approved for MVP Development |
| **Target Audience** | Developers, Architects, QA, Product Owner |
| **Implementation Window** | Approximately 1 week (MVP / Proof of Concept) |

---

## 2. Project Overview

InfyBank360 is a secure digital banking platform that gives customers a single interface to manage accounts, move money, pay bills, apply for loans, review transactions, and raise service requests — while giving bank employees the tools to onboard customers, process requests, monitor activity, and manage compliance. 

This iteration of the platform adopts a deliberately consolidated microservice architecture (7 services) to balance independent scaling and clear ownership without over-fragmenting the platform. It leverages event-driven communication for notifications, fraud, and audit, alongside synchronous REST for direct request/response flows.

---

## 3. Project Objectives

The project must deliver a working MVP that demonstrates the following capabilities:

1. **Consolidated Microservices:** 7 independently deployable services with clear business boundaries.
2. **Hybrid Communication:** Synchronous REST for request/response flows; Asynchronous Pub/Sub (Message Broker) for event-driven notifications, fraud, and audit.
3. **Multi-Database Strategy:** Relational databases for financial data, Document stores for flexible customer data, and In-memory caches for fast reads.
4. **Unified Entry Point:** A combined Identity and API Gateway service handling authentication, RBAC, and routing.
5. **Containerized Delivery:** All services and infrastructure packaged as containers and managed via a container orchestrator (Docker Compose for the MVP).
6. **Observability:** Centralized logging, metrics, and tracing across all services.

---

## 4. Scope

### 4.1 In Scope

| Area | In Scope Details |
|------|----------|
| **Customer channels** | Web and mobile access to all banking services (APIs only for MVP). |
| **Employee channels** | Onboarding, service-request processing, activity monitoring, compliance. |
| **Core banking** | Accounts, transfers, transaction history, ledger posting. |
| **Payments** | Bill pay, biller management, payment status tracking. |
| **Lending** | Loan application capture, eligibility check, approval workflow, disbursement tracking. |
| **Security** | Authentication, authorization (RBAC), audit logging. |
| **Notifications** | Event-driven email, SMS, and push alerts for banking events. |

### 4.2 Out of Scope

The following are explicitly **excluded** from the MVP to maintain the one-week implementation timeline:

- Multi-Factor Authentication (MFA)
- Know Your Customer (KYC) external integrations
- Session timeout and advanced session management
- Data encryption (at rest and in transit)
- Real payment-gateway integration (mocked for MVP)
- Real-time alerting (fraud/alerting is batch/offline or near-real-time via events)

---

## 5. Actors

| # | Actor | Description | Primary Interactions |
|---|---|---|---|
| 1 | **Customer** | End-user who holds accounts and uses banking services. | Registers, logs in, views accounts/balances, transfers money, pays bills, applies for loans, raises service requests, receives notifications. |
| 2 | **Bank Employee** | Staff member supporting customers and processing operations. | Logs in, onboards customers, processes service requests, monitors accounts, approves/rejects loans. |
| 3 | **Compliance Officer** | Staff member responsible for regulatory and security oversight. | Reviews audit trails, investigates fraud detection results, generates regulatory reports. |
| 4 | **Administrator** | Privileged user managing system configuration. | Manages system configuration, roles, access, and fraud rules. |

---

## 6. System Overview

### 6.1 High-Level Architecture
- **Pattern:** Microservices with a shared API gateway. Event-driven communication (Pub/Sub) is used for notifications, fraud, and audit. Synchronous REST calls are used for direct request/response flows.
- **Data:** Database-per-service. No shared database across services. Utilizes Relational, Document, and Cache stores.
- **Integration:** REST for synchronous calls; Message Broker for asynchronous events.
- **Cross-cutting:** Centralized logging, metrics, tracing, and a shared config/secrets layer.

### 6.2 Logical Flow
```text
[ Clients (Web/Mobile) ] 
       │
       ▼
[ Identity & API Gateway ] ──(Validates Tokens)──> [ All Services ]
       │
       ├─(Sync REST)──> [ Customer & Account ] <──> [ Document Store / Relational ]
       ├─(Sync REST)──> [ Transaction ] ──(Sync)──> [ Payment & Bill Pay ]
       ├─(Sync REST)──> [ Loan Origination ] ──(Sync)──> [ Transaction ]
       │
       ▼ (Async Pub/Sub Events)
[ Message Broker ]
       │
       ├─(Subscribe)──> [ Fraud & Compliance ] ──> [ Relational / Audit DB ]
       └─(Subscribe)──> [ Notification Service ] ──> [ Email/SMS/Push Mocks ]
```

---

## 7. Functional Areas

### 7.1 Authentication & Identity (FR-7)
- Customer, employee, and admin login.
- JWT-based authentication and Role-Based Access Control (RBAC).
- Single entry point via the API Gateway for all client traffic.

### 7.2 Customer Management (FR-1)
- Customer onboarding and profile management.
- Profile data stored in a Document Store for structural flexibility.
- Employee-driven onboarding workflow.

### 7.3 Account Management (FR-1)
- Account creation, lifecycle management, and balance tracking.
- Balances cached in an In-Memory Cache for low-latency reads.

### 7.4 Transaction Processing (FR-2)
- Fund transfers, ledger posting, and balance management.
- Transaction history and statement generation.
- Publishes domain events (e.g., `TransferCompleted`) to the message broker.

### 7.5 Payments & Bill Pay (FR-3)
- Biller management and bill payment initiation.
- Payment status tracking.
- Payment gateway integration is mocked.

### 7.6 Loan Origination (FR-4)
- Loan application capture and eligibility checks.
- Approval workflow and disbursement tracking.
- Disbursement triggers ledger postings via the Transaction Service.

### 7.7 Service Requests (FR-1)
- Customer raises service requests (stored in Document Store).
- Employee processes and updates request status.

### 7.8 Notification Management (FR-6)
- Event-driven delivery of Email, SMS, and Push notifications.
- Consumes domain events from money-movement services.
- No direct business logic; purely a delivery mechanism.

### 7.9 Fraud Monitoring (FR-5)
- Fraud detection rules (batch/offline evaluation).
- Consumes transaction and payment events to evaluate rules.
- Generates fraud alerts for compliance review.

### 7.10 Audit and Compliance (FR-5)
- Audit trail for all money-movement and data-access events.
- Regulatory reporting hooks.
- Combined with Fraud & Compliance as both are event-driven oversight functions.

---

## 8. User Stories

### US-001 — Customer Registration & Onboarding
- **Actor:** Customer / Bank Employee
- **Story:** As a user, I want to register or be onboarded so that I can access banking services.
- **Acceptance Criteria:** Profile is saved in the Document Store; initial account is created; credentials are hashed.
- **Priority:** P0

### US-002 — Login and Token Issuance
- **Actor:** All Users
- **Story:** As a user, I want to log in and receive a JWT so that I can access protected APIs.
- **Acceptance Criteria:** Valid credentials return a JWT containing user ID and role. Invalid credentials return a generic error.
- **Priority:** P0

### US-003 — View Account Balance
- **Actor:** Customer
- **Story:** As a customer, I want to view my account balance quickly.
- **Acceptance Criteria:** Balance is retrieved from the In-Memory Cache for low latency. Falls back to DB if cache misses.
- **Priority:** P0

### US-004 — Transfer Funds
- **Actor:** Customer
- **Story:** As a customer, I want to transfer money to another account.
- **Acceptance Criteria:** Ledger is updated atomically; `TransferCompleted` event is published to the broker; balances are updated in cache.
- **Priority:** P0

### US-005 — View Transaction History
- **Actor:** Customer
- **Story:** As a customer, I want to view my past transactions.
- **Acceptance Criteria:** History is paginated and retrieved efficiently.
- **Priority:** P0

### US-006 — Pay a Bill
- **Actor:** Customer
- **Story:** As a customer, I want to pay a utility bill.
- **Acceptance Criteria:** Payment is processed via mocked gateway; account is debited; `BillPaid` event is published.
- **Priority:** P1

### US-007 — Apply for a Loan
- **Actor:** Customer
- **Story:** As a customer, I want to apply for a loan.
- **Acceptance Criteria:** Application is saved with PENDING status; `LoanApplied` event is published.
- **Priority:** P1

### US-008 — Approve Loan and Disburse
- **Actor:** Bank Employee
- **Story:** As an employee, I want to approve a loan and trigger disbursement.
- **Acceptance Criteria:** Loan status changes to APPROVED; Transaction Service is called synchronously to credit the customer's account.
- **Priority:** P1

### US-009 — Raise a Service Request
- **Actor:** Customer
- **Story:** As a customer, I want to raise a service request.
- **Acceptance Criteria:** Request is saved in the Document Store with OPEN status.
- **Priority:** P1

### US-010 — Process Service Request
- **Actor:** Bank Employee
- **Story:** As an employee, I want to update a service request status.
- **Acceptance Criteria:** Status transitions to IN_PROGRESS or RESOLVED; `ServiceRequestUpdated` event is published.
- **Priority:** P1

### US-011 — Receive Transaction Notification
- **Actor:** Customer (System)
- **Story:** As a customer, I want to receive a notification when a transaction occurs.
- **Acceptance Criteria:** Notification Service consumes the `TransferCompleted` event and logs/sends a mock notification.
- **Priority:** P1

### US-012 — Fraud Alert on High-Value Transfer
- **Actor:** Compliance Officer (System)
- **Story:** As the system, I want to flag high-value transfers for review.
- **Acceptance Criteria:** Fraud Service consumes transfer event, evaluates rule, and creates a fraud alert if threshold is exceeded.
- **Priority:** P1

### US-013 — Review Audit Trail
- **Actor:** Compliance Officer
- **Story:** As a compliance officer, I want to review the audit trail of money movements.
- **Acceptance Criteria:** Audit logs are queryable by date, user, and action type.
- **Priority:** P1

### US-014 — Configure Fraud Rules
- **Actor:** Administrator
- **Story:** As an admin, I want to configure fraud detection thresholds.
- **Acceptance Criteria:** Admin can update the high-value transfer threshold via API.
- **Priority:** P2

### US-015 — View Customer Profile
- **Actor:** Customer
- **Story:** As a customer, I want to view and update my profile details.
- **Acceptance Criteria:** Profile is fetched from the Document Store; updates are persisted.
- **Priority:** P1

---

## 9. Functional Requirements

### 9.1 FR-1: Account & Customer Management
- **FR-1.1:** System shall onboard customers and store flexible profile data in a Document Store.
- **FR-1.2:** System shall create and manage bank accounts linked to customers.
- **FR-1.3:** System shall allow employees to process and update customer service requests.

### 9.2 FR-2: Transaction Processing
- **FR-2.1:** System shall process fund transfers with atomic ledger posting.
- **FR-2.2:** System shall maintain account balances and update the In-Memory Cache on every transaction.
- **FR-2.3:** System shall publish domain events for all completed transactions.

### 9.3 FR-3: Payments & Bill Pay
- **FR-3.1:** System shall manage billers and process bill payments.
- **FR-3.2:** System shall mock the external payment gateway for MVP.
- **FR-3.3:** System shall publish payment events upon successful bill payment.

### 9.4 FR-4: Loan Origination
- **FR-4.1:** System shall capture loan applications and perform basic eligibility checks.
- **FR-4.2:** System shall manage the approval workflow (PENDING, APPROVED, REJECTED).
- **FR-4.3:** System shall trigger disbursement via the Transaction Service upon approval.

### 9.5 FR-5: Fraud & Compliance
- **FR-5.1:** System shall consume money-movement events to evaluate fraud rules.
- **FR-5.2:** System shall maintain a comprehensive audit trail for all financial and data-access events.
- **FR-5.3:** System shall provide hooks for regulatory reporting.

### 9.6 FR-6: Notification
- **FR-6.1:** System shall consume domain events to trigger Email, SMS, and Push notifications.
- **FR-6.2:** System shall manage notification templates and delivery logs.

### 9.7 FR-7: Identity & Access
- **FR-7.1:** System shall authenticate users and issue JWTs.
- **FR-7.2:** System shall enforce RBAC across all API endpoints.
- **FR-7.3:** System shall route all client traffic through the API Gateway.

---

## 10. Business Rules

1. **BR-001:** Customer email must be unique.
2. **BR-002:** Passwords must be hashed (BCrypt) before storage.
3. **BR-003:** Only authenticated users with valid JWTs can access protected APIs.
4. **BR-004:** Customers can only access their own accounts, transactions, and data.
5. **BR-005:** Account balances must be updated in the cache immediately after a successful transaction.
6. **BR-006:** Transfers must be atomically posted to the ledger; no partial updates allowed.
7. **BR-007:** Loan disbursements must strictly follow the approval workflow.
8. **BR-008:** All money-movement events must be published to the message broker for audit and fraud evaluation.
9. **BR-009:** Fraud rules are evaluated asynchronously; they do not block the primary transaction flow in the MVP.
10. **BR-010:** Service requests must follow the state machine: OPEN → IN_PROGRESS → RESOLVED.

---

## 11. Non-Functional Requirements

- **Security:** Authentication, secrets management, least-privilege access. (MFA and encryption in transit/at rest out of scope).
- **Compliance:** Audit trail for all money-movement and data-access events; regulatory reporting hooks.
- **Availability:** High availability with no single point of failure (architected for, simulated via Docker Compose in MVP).
- **Performance:** Low-latency responses for balance and transaction queries (achieved via In-Memory Cache).
- **Scalability:** Independent scaling of high-load services (transactions, payments, notifications).
- **Observability:** Centralized logging, metrics, and tracing across all services.
- **Maintainability:** Independently deployable services, automated CI/CD, versioned APIs.

---

## 12. Technology Stack

| Concern | Technology | How it is used in InfyBank360 |
|---------|------------|----------------|
| **Service APIs** | REST/JSON over HTTP | Every service exposes REST endpoints; Gateway routes traffic. Sync calls for req/res flows. |
| **Async events** | Message Broker (RabbitMQ) | Pub/Sub for domain events (transfers, payments). Decouples money-movement from Fraud/Notification. |
| **Financial data** | Relational DB (PostgreSQL) | Ledger, balances, transactions, accounts. Ensures transactional integrity. |
| **Flexible data** | Document Store (MongoDB) | Customer profiles and service-request records. |
| **Fast reads** | In-memory cache (Redis) | Balance and transaction-history lookups for low-latency NFR. |
| **Packaging** | Containers (Docker) | Each of the 7 services shipped as a container image. |
| **Runtime** | Container Orchestrator | Docker Compose for MVP local runtime; Kubernetes-ready design. |
| **Security** | API Gateway + JWT + RBAC | Gateway validates tokens; services enforce roles. |
| **Observability** | Centralized Stack | Structured logs, metrics (Prometheus), tracing (Zipkin/Jaeger). |
| **Config & secrets** | Shared Config Store | Spring Cloud Config or centralized environment variables. |
| **Delivery** | CI/CD Pipeline | GitHub Actions for automated build, test, and containerization. |

---

## 13. Microservices Requirements

The domain is consolidated to **7 services**.

### 13.1 Identity & API Gateway Service
- **Responsibility:** Authentication, RBAC, single entry point for all clients.
- **Data Owned:** Users, roles, JWT signing keys.
- **Dependencies:** None (Core infrastructure).
- **Mandatory:** ✅ Yes (P0)

### 13.2 Customer & Account Service
- **Responsibility:** Customer onboarding, profiles, accounts, service requests.
- **Data Owned:** Customer profiles (MongoDB), Accounts (PostgreSQL), Service Requests (MongoDB).
- **Dependencies:** Identity Service (auth validation).
- **Mandatory:** ✅ Yes (P0)

### 13.3 Transaction Service
- **Responsibility:** Ledger, fund transfers, balances, transaction history.
- **Data Owned:** Transactions, Ledger (PostgreSQL), Balances (Redis).
- **Dependencies:** Payment Service (sync for settlement).
- **Mandatory:** ✅ Yes (P0)

### 13.4 Payment & Bill Pay Service
- **Responsibility:** Billers, bill payments, payment tracking.
- **Data Owned:** Payments, Billers (PostgreSQL).
- **Dependencies:** Transaction Service (sync for account debit).
- **Mandatory:** ⚠️ P1 (Simplifiable for MVP)

### 13.5 Fraud & Compliance Service
- **Responsibility:** Fraud rules, audit logging, regulatory reports.
- **Data Owned:** Fraud alerts, Audit logs, Rules (PostgreSQL/MongoDB).
- **Dependencies:** Message Broker (consumes events from all money-movement services).
- **Mandatory:** ⚠️ P1 (Core logging is P0, advanced fraud is P1)

### 13.6 Notification Service
- **Responsibility:** Email, SMS, push delivery and templates.
- **Data Owned:** Notification logs, Templates (PostgreSQL).
- **Dependencies:** Message Broker (consumes events from all services).
- **Mandatory:** ⚠️ P1 (Simplifiable to DB persistence for MVP)

### 13.7 Loan Origination Service
- **Responsibility:** Loan application, eligibility, approval, disbursement.
- **Data Owned:** Loans, Disbursements (PostgreSQL).
- **Dependencies:** Customer & Account (sync validation), Transaction Service (sync disbursement).
- **Mandatory:** ⚠️ P1

---

## 14. Data and Database Requirements

- **Database-per-service:** No service shares a database schema with another.
- **Relational (PostgreSQL):** Used for financial data requiring strict ACID compliance (Accounts, Transactions, Payments, Loans).
- **Document Store (MongoDB):** Used for flexible, semi-structured data (Customer Profiles, Service Requests).
- **In-Memory Cache (Redis):** Used for high-read, low-latency data (Account Balances, recent Transaction History).
- **Event Store / Message Broker:** RabbitMQ handles the asynchronous event streams.

---

## 15. API Requirements

High-level REST capabilities exposed via the API Gateway:

- **Auth:** `POST /auth/login`, `POST /auth/register`
- **Customers:** `GET /customers/{id}`, `PUT /customers/{id}`
- **Accounts:** `GET /accounts/{id}/balance`, `GET /accounts/customer/{customerId}`
- **Transactions:** `POST /transactions/transfer`, `GET /transactions/account/{accountId}`
- **Payments:** `POST /payments/bill`, `GET /payments/{id}`
- **Loans:** `POST /loans`, `PUT /loans/{id}/approve`
- **Service Requests:** `POST /service-requests`, `PUT /service-requests/{id}/status`

---

## 16. Security Requirements

- **Authentication:** JWT issued by Identity Service, validated by API Gateway.
- **Authorization:** RBAC enforced at the Gateway and Service levels (CUSTOMER, EMPLOYEE, COMPLIANCE, ADMIN).
- **Secrets:** Database credentials and broker connections managed via shared config/environment variables.
- **Data Protection:** Passwords hashed. (Full encryption at rest/in transit is out of scope for MVP).

---

## 17. Inter-Service Communication

- **Synchronous REST:** 
  - Gateway → Services
  - Transaction → Payment (for settlement)
  - Loan → Customer/Account (for validation)
  - Loan → Transaction (for disbursement)
- **Asynchronous Pub/Sub (Message Broker):**
  - Transaction → publishes `TransferCompleted`
  - Payment → publishes `BillPaid`
  - Loan → publishes `LoanDisbursed`
  - Fraud & Notification → subscribe to all financial events.

---

## 18. Error Handling

Standardized JSON error response for all APIs:
```json
{
  "timestamp": "2026-10-07T10:00:00Z",
  "status": 400,
  "errorCode": "INSUFFICIENT_BALANCE",
  "message": "The source account does not have sufficient funds.",
  "path": "/api/v1/transactions/transfer"
}
```

---

## 19. Logging and Auditing

- **Application Logging:** Structured JSON logs emitted to standard output, collected by the centralized observability stack.
- **Audit Logging:** The Fraud & Compliance Service consumes domain events and persists a permanent audit trail of WHO, WHAT, WHEN, and OUTCOME for all financial operations.

---

## 20. MVP Priorities

- **P0 (Must Have for 1-Week MVP):** Identity & Gateway, Customer & Account (basic), Transaction (basic transfer), Relational DB setup, Docker Compose runtime.
- **P1 (Important, simplify if needed):** Payment, Loan, Notification (DB only), Fraud (basic rules), Redis Cache, MongoDB integration.
- **P2 (Future):** Real payment gateway, MFA, full ELK observability stack, Kubernetes deployment.

---

## 21. Assumptions and Constraints

- **Constraint:** Must be implementable in **ONE WEEK**. Therefore, infrastructure (MongoDB, Redis, RabbitMQ) will be run locally via **Docker Compose**, not production Kubernetes.
- **Assumption:** Start with 7 services; split further only when load demands it.
- **Assumption:** External payment gateway and financial-institution integrations are designed now, implemented later (mocked).
- **Assumption:** Regulatory requirements are region-agnostic and must be finalized with the compliance team later.

---

## 22. Future Enhancements

- Multi-Factor Authentication (MFA)
- KYC/AML external integrations
- Data encryption at rest and in transit
- Real payment-gateway integration
- Real-time fraud alerting and blocking
- Production Kubernetes deployment and advanced CI/CD

---

## 23. Traceability Matrix

| User Story | Functional Requirement | Microservice | Priority |
|---|---|---|---|
| US-001 — Registration | FR-1.1, FR-7.1 | Identity, Customer & Account | P0 |
| US-002 — Login | FR-7.1, FR-7.2 | Identity & API Gateway | P0 |
| US-003 — View Balance | FR-2.2 | Customer & Account (Redis) | P0 |
| US-004 — Transfer Funds | FR-2.1, FR-2.3 | Transaction, Payment | P0 |
| US-005 — Txn History | FR-2.1 | Transaction | P0 |
| US-006 — Pay Bill | FR-3.1, FR-3.3 | Payment, Transaction | P1 |
| US-007 — Apply Loan | FR-4.1 | Loan Origination | P1 |
| US-008 — Approve Loan | FR-4.2, FR-4.3 | Loan, Transaction | P1 |
| US-009 — Raise SR | FR-1.3 | Customer & Account (Mongo) | P1 |
| US-010 — Process SR | FR-1.3 | Customer & Account | P1 |
| US-011 — Notification | FR-6.1 | Notification (via Broker) | P1 |
| US-012 — Fraud Alert | FR-5.1 | Fraud & Compliance (via Broker)| P1 |
| US-013 — Audit Trail | FR-5.2 | Fraud & Compliance | P1 |
| US-014 — Config Rules | FR-5.1 | Fraud & Compliance | P2 |
| US-015 — View Profile | FR-1.1 | Customer & Account (Mongo) | P1 |

---

**End of Requirement Specification — InfyBank360 v2.0 (Fresh Architecture)**