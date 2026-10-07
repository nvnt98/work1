# InfyBank360 - Requirement Specification

> **Note to the reader (Navneet):** This document describes **what** the InfyBank360 system must do and **why** — it intentionally does **not** describe **how** it will be implemented. The implementation details (Java code, Spring annotations, database schemas, configuration files, etc.) will be covered later in the technical specification (`spec.md`). Treat this document as the "contract" that the development team agrees on before writing any code.

---

## 1. Document Information

| Field | Value |
|---|---|
| **Project Name** | InfyBank360 |
| **Document Title** | Requirement Specification |
| **Document Version** | 1.0 |
| **Date** | October 07, 2026 |
| **Author** | Business Analysis & Architecture Team |
| **Status** | Draft — Approved for MVP Development |
| **Target Audience** | Developers, Architects, QA, Product Owner |
| **Implementation Window** | Approximately 1 week (MVP) |

---

## 2. Project Overview

**InfyBank360** is a secure, educational digital banking platform designed to demonstrate modern microservice architecture using Spring Boot. It gives customers convenient access to everyday banking services (accounts, transfers, bill payments, loans, service requests) and gives bank employees the tools to support customers, process requests, and monitor activity.

The platform is intentionally **simplified** — it is **not** a production banking system. It is a learning-oriented MVP that showcases:

- Microservice decomposition
- REST API design
- Service-to-service communication
- Database-per-service pattern
- Authentication, authorization, and auditability
- Basic fraud rules and notifications

External systems (payment gateways, SMS/email providers, credit bureaus) are **mocked** to keep the scope realistic for a one-week implementation.

---

## 3. Project Objectives

The project must deliver a working MVP that demonstrates the following capabilities:

1. **Spring Boot 3.x microservices** running on Java 21.
2. **REST APIs** exposed through a single API Gateway.
3. **Service discovery** via Eureka so services find each other dynamically.
4. **Database-per-service** using PostgreSQL (each service owns its data).
5. **JWT-based authentication** with role-based authorization (CUSTOMER, EMPLOYEE, ADMIN).
6. **Synchronous inter-service communication** using REST (RestClient/WebClient).
7. **Input validation** and **global exception handling** with consistent error responses.
8. **Transaction processing** that maintains consistent account balances.
9. **Basic rule-based fraud detection** (thresholds, account status checks).
10. **Notifications** persisted in the database (no real email/SMS required).
11. **Audit logging** for important banking operations.
12. **OpenAPI/Swagger documentation** for every service.
13. **Automated unit tests** using JUnit 5 and Mockito.

The MVP must be **implementable in approximately one week** by a small team. Simplicity and clarity are prioritized over feature volume.

---

## 4. Scope

### 4.1 In Scope

- Customer registration and login
- Employee and administrator login
- JWT-based authentication and role-based authorization
- Customer profile management
- Bank account creation, viewing, and balance checks
- Money transfers between accounts (internal)
- Transaction history viewing
- Bill payments (electricity, mobile, internet, water) with a mocked payment gateway
- Loan application submission, viewing, and employee/admin approval or rejection
- Service request creation and status updates by employees
- Notification generation and history (stored in DB)
- Basic rule-based fraud monitoring
- Audit logging for key operations
- API Gateway for unified entry point
- Eureka service discovery
- PostgreSQL persistence
- REST-based inter-service communication
- OpenAPI documentation
- Unit tests

### 4.2 Out of Scope

The following are explicitly **excluded** from the MVP. They may be considered as future enhancements.

- Real money movement or integration with a core banking system
- Real payment gateway integration (e.g., Stripe, Razorpay)
- Real SMS or email providers (Twilio, SendGrid, etc.)
- Advanced fraud detection using machine learning
- Credit bureau integration
- KYC/AML external integrations
- Kubernetes, AWS, or advanced cloud infrastructure
- Event streaming platforms (Kafka, RabbitMQ) — unless strictly justified later
- High-frequency trading or complex regulatory reporting
- Production-grade disaster recovery and multi-region deployment
- Multi-currency support
- Mobile or web UI (APIs only for the MVP)

---

## 5. Actors

| # | Actor | Description | Primary Interactions |
|---|---|---|---|
| 1 | **Customer** | An end-user who holds one or more bank accounts with InfyBank360. | Registers, logs in, views accounts/balances, transfers money, pays bills, applies for loans, raises service requests, views notifications. |
| 2 | **Bank Employee** | A staff member who supports customers and processes operational requests. | Logs in, onboards customers, views customer and account information, approves/rejects loans, updates service request status, monitors transactions. |
| 3 | **Bank Administrator** | A privileged staff member with elevated access for oversight and compliance. | Performs all employee actions plus system oversight, audit review, and configuration of fraud thresholds. |
| 4 | **External System** | A placeholder for third-party integrations (payment gateway, notification provider, external financial institutions). | In the MVP, these are **mocked** inside the respective services. No real external calls are made. |

---

## 6. System Overview

InfyBank360 is composed of multiple independently deployable microservices behind a single **API Gateway**. Services register themselves with **Eureka** for discovery. Customers and employees interact only through the gateway. Services communicate with each other over **synchronous REST** calls. Each service owns its own **PostgreSQL database** and never reads another service's database directly.

A high-level view of the logical components:

```
[ Customer / Employee / Admin ]
              |
              v
        [ API Gateway ]  ----> [ Eureka Service Registry ]
              |
   +----------+----------+----------+----------+----------+
   |          |          |          |          |          |
[Auth]    [Customer]  [Account] [Transaction] [Payment] [Loan]
   |          |          |          |          |          |
   +----------+----------+----------+----------+----------+
              |                                     |
        [Notification]  <---------------------------+
              |
        [PostgreSQL DB per service]
```

This structure keeps responsibilities clear, allows services to evolve independently, and mirrors real-world banking architecture at an educational scale.

---

## 7. Functional Areas

### 7.1 Authentication

- Customer registration with validated personal information.
- Customer, employee, and administrator login issuing a JWT.
- Role-based authorization (CUSTOMER, EMPLOYEE, ADMIN).
- Passwords hashed using BCrypt; never stored in plain text.
- Invalid login attempts handled with clear error messages.
- JWT validated on every protected request.

### 7.2 Customer Management

- Maintain customer profile (name, email, phone, address, status).
- Customer onboarding performed by employees/admins.
- Customers can view and update their own profile.
- Employees/admins can look up customers.
- Customer status: ACTIVE, INACTIVE, BLOCKED.

### 7.3 Account Management

- Create bank accounts linked to a customer.
- View account details and current balance.
- View all accounts belonging to a customer.
- Account status: ACTIVE, INACTIVE, BLOCKED, CLOSED.
- Basic validation (e.g., account exists, is active, belongs to the caller).

### 7.4 Transaction Management

- Internal money transfers between accounts.
- Debit and credit operations that update balances atomically.
- Transaction history per account.
- Transaction status: SUCCESS, FAILED, PENDING, FLAGGED.
- Validation: sufficient balance, active accounts, positive amount, matching source and destination.
- Insufficient balance and invalid account scenarios handled gracefully.

### 7.5 Payment Management

- Bill payments for categories: Electricity, Mobile, Internet, Water.
- Payment status: SUCCESS, FAILED, PENDING.
- Payment history per customer.
- Payment gateway is **mocked** — payments succeed deterministically for valid requests.

### 7.6 Loan Management

- Customers submit loan applications (amount, tenure, purpose).
- Customers view their own loan applications.
- Employees/admins approve or reject loan applications.
- Loan status: PENDING, APPROVED, REJECTED.

### 7.7 Service Requests

- Customers raise service requests with a category (e.g., Card Block, Statement Request, Account Update).
- Customers view their own service requests.
- Employees update service request status.
- Status: OPEN, IN_PROGRESS, RESOLVED.

### 7.8 Notification Management

- Notifications generated for important events (transaction, payment, loan status change, service request update).
- Notifications persisted in the database (no real email/SMS).
- Customers can view their notification history.

### 7.9 Fraud Monitoring

Simple **rule-based** fraud detection only. No machine learning.

Example rules:
- Transactions above a configurable threshold are flagged.
- Transactions from inactive or blocked accounts are rejected.
- Transactions exceeding available balance are rejected.
- Multiple rapid transfers from the same account may be flagged.

### 7.10 Audit and Compliance

Audit records are created for important operations:
- Login
- Account creation
- Money transfer
- Payment
- Loan submission / approval / rejection
- Service request updates

Audit records capture **who**, **what**, **when**, and **outcome** (success/failure). Detailed regulatory reporting is out of scope.

---

## 8. User Stories

### US-001 — Customer Registration
- **Actor:** Customer
- **Story:** As a customer, I want to register with my personal details so that I can access digital banking services.
- **Acceptance Criteria:**
  - Required fields (name, email, phone, password) must be provided.
  - Email must be valid and unique.
  - Password must meet minimum security requirements.
  - Password must not be stored in plain text.
  - Successful registration returns a confirmation response.
- **Priority:** P0

### US-002 — Customer Login
- **Actor:** Customer
- **Story:** As a customer, I want to log in with my credentials so that I can access my accounts.
- **Acceptance Criteria:**
  - Valid credentials return a JWT.
  - Invalid credentials return a clear error.
  - The JWT contains the user's role.
- **Priority:** P0

### US-003 — Employee / Admin Login
- **Actor:** Bank Employee / Bank Administrator
- **Story:** As an employee or admin, I want to log in so that I can perform operational tasks.
- **Acceptance Criteria:**
  - Valid credentials return a JWT with EMPLOYEE or ADMIN role.
  - Invalid credentials return a clear error.
- **Priority:** P0

### US-004 — View Account Balance
- **Actor:** Customer
- **Story:** As a customer, I want to view my account balance so that I know how much money I have.
- **Acceptance Criteria:**
  - Only the account owner can view the balance.
  - The response includes the current balance and account status.
- **Priority:** P0

### US-005 — Transfer Money
- **Actor:** Customer
- **Story:** As a customer, I want to transfer money to another account so that I can pay someone.
- **Acceptance Criteria:**
  - Source and destination accounts must exist and be active.
  - Transfer amount must be greater than zero.
  - Source account must have sufficient balance.
  - On success, balances are updated and a transaction record is created.
  - On failure, balances remain unchanged and a clear error is returned.
- **Priority:** P0

### US-006 — View Transaction History
- **Actor:** Customer
- **Story:** As a customer, I want to view my transaction history so that I can review my activity.
- **Acceptance Criteria:**
  - Only the account owner can view the history.
  - Transactions are returned in reverse chronological order.
- **Priority:** P0

### US-007 — Pay a Bill
- **Actor:** Customer
- **Story:** As a customer, I want to pay a utility bill so that I can settle my dues.
- **Acceptance Criteria:**
  - Supported categories: Electricity, Mobile, Internet, Water.
  - Sufficient balance is required.
  - The payment gateway is mocked and returns a deterministic result.
  - A payment record and a notification are created on success.
- **Priority:** P1

### US-008 — Apply for a Loan
- **Actor:** Customer
- **Story:** As a customer, I want to apply for a loan so that I can borrow money.
- **Acceptance Criteria:**
  - Loan application captures amount, tenure, and purpose.
  - Initial status is PENDING.
  - A notification is generated on submission.
- **Priority:** P1

### US-009 — Approve or Reject a Loan
- **Actor:** Bank Employee / Bank Administrator
- **Story:** As an employee or admin, I want to approve or reject a loan application so that the customer's request is resolved.
- **Acceptance Criteria:**
  - Only EMPLOYEE or ADMIN roles can perform this action.
  - Status transitions from PENDING to APPROVED or REJECTED.
  - A notification is sent to the customer.
- **Priority:** P1

### US-010 — Raise a Service Request
- **Actor:** Customer
- **Story:** As a customer, I want to raise a service request so that the bank can help me with an issue.
- **Acceptance Criteria:**
  - Request includes a category and description.
  - Initial status is OPEN.
  - A notification is generated.
- **Priority:** P1

### US-011 — Update Service Request Status
- **Actor:** Bank Employee
- **Story:** As an employee, I want to update a service request's status so that the customer is informed of progress.
- **Acceptance Criteria:**
  - Only EMPLOYEE or ADMIN roles can update status.
  - Valid transitions: OPEN → IN_PROGRESS → RESOLVED.
  - A notification is sent to the customer.
- **Priority:** P1

### US-012 — View Notifications
- **Actor:** Customer
- **Story:** As a customer, I want to view my notifications so that I am informed about important events.
- **Acceptance Criteria:**
  - Only the customer's own notifications are returned.
  - Notifications include type, message, and timestamp.
- **Priority:** P1

### US-013 — View Customer Profile
- **Actor:** Customer
- **Story:** As a customer, I want to view my profile so that I can verify my details.
- **Acceptance Criteria:**
  - Only the customer's own profile is accessible.
  - Profile includes name, email, phone, address, and status.
- **Priority:** P1

### US-014 — View Account Details
- **Actor:** Customer
- **Story:** As a customer, I want to view my account details so that I can confirm account information.
- **Acceptance Criteria:**
  - Only the account owner can view the details.
  - Response includes account number, type, status, and balance.
- **Priority:** P0

### US-015 — High-Value Transaction Flagging
- **Actor:** System (Fraud Monitoring)
- **Story:** As the system, I want to flag transactions above a configurable threshold so that suspicious activity can be reviewed.
- **Acceptance Criteria:**
  - Transactions above the configured threshold are marked with status FLAGGED.
  - The transaction still completes unless other rules reject it.
  - A notification is generated for the flagged transaction.
- **Priority:** P1

### US-016 — Audit Logging of Key Operations
- **Actor:** System (Audit)
- **Story:** As the system, I want to log key operations so that administrators can trace activity.
- **Acceptance Criteria:**
  - Audit records are created for login, account creation, transfer, payment, loan actions, and service request updates.
  - Each record includes actor, action, timestamp, and outcome.
- **Priority:** P1

### US-017 — Employee Onboards a Customer
- **Actor:** Bank Employee
- **Story:** As an employee, I want to onboard a customer so that they can start using banking services.
- **Acceptance Criteria:**
  - Only EMPLOYEE or ADMIN roles can onboard.
  - Customer is created with ACTIVE status.
  - An initial account may be created as part of onboarding.
- **Priority:** P1

### US-018 — View Payment History
- **Actor:** Customer
- **Story:** As a customer, I want to view my payment history so that I can track past bill payments.
- **Acceptance Criteria:**
  - Only the customer's own payments are returned.
  - Payments include category, amount, status, and timestamp.
- **Priority:** P1

---

## 9. Functional Requirements

### 9.1 Authentication (FR-AUTH)

| ID | Description | Actor | Priority |
|---|---|---|---|
| FR-AUTH-001 | The system shall allow a customer to register with name, email, phone, and password. | Customer | P0 |
| FR-AUTH-002 | The system shall reject registration if the email is invalid or already exists. | Customer | P0 |
| FR-AUTH-003 | The system shall hash passwords using BCrypt before storage. | System | P0 |
| FR-AUTH-004 | The system shall issue a JWT upon successful login for customers, employees, and admins. | All | P0 |
| FR-AUTH-005 | The system shall reject invalid login attempts with a generic error message. | All | P0 |
| FR-AUTH-006 | The system shall include the user's role inside the JWT. | System | P0 |

### 9.2 Customer Management (FR-CUST)

| ID | Description | Actor | Priority |
|---|---|---|---|
| FR-CUST-001 | The system shall allow customers to view their own profile. | Customer | P1 |
| FR-CUST-002 | The system shall allow customers to update their own profile (limited fields). | Customer | P1 |
| FR-CUST-003 | The system shall allow employees/admins to onboard a new customer. | Employee/Admin | P1 |
| FR-CUST-004 | The system shall allow employees/admins to look up customers by ID or email. | Employee/Admin | P1 |
| FR-CUST-005 | The system shall maintain customer status (ACTIVE, INACTIVE, BLOCKED). | System | P0 |

### 9.3 Account Management (FR-ACC)

| ID | Description | Actor | Priority |
|---|---|---|---|
| FR-ACC-001 | The system shall allow creation of a bank account linked to a customer. | Customer/Employee | P0 |
| FR-ACC-002 | The system shall allow customers to view their own accounts. | Customer | P0 |
| FR-ACC-003 | The system shall allow customers to view the balance of their own accounts. | Customer | P0 |
| FR-ACC-004 | The system shall maintain account status (ACTIVE, INACTIVE, BLOCKED, CLOSED). | System | P0 |
| FR-ACC-005 | The system shall reject operations on non-active accounts. | System | P0 |

### 9.4 Transaction Management (FR-TXN)

| ID | Description | Actor | Priority |
|---|---|---|---|
| FR-TXN-001 | The system shall allow internal money transfers between accounts. | Customer | P0 |
| FR-TXN-002 | The system shall reject transfers where the amount is zero or negative. | System | P0 |
| FR-TXN-003 | The system shall reject transfers where the source account has insufficient balance. | System | P0 |
| FR-TXN-004 | The system shall debit the source and credit the destination atomically. | System | P0 |
| FR-TXN-005 | The system shall record every transaction with a unique ID, status, and timestamp. | System | P0 |
| FR-TXN-006 | The system shall allow customers to view transaction history for their own accounts. | Customer | P0 |

### 9.5 Payment Management (FR-PAY)

| ID | Description | Actor | Priority |
|---|---|---|---|
| FR-PAY-001 | The system shall allow bill payments in categories: Electricity, Mobile, Internet, Water. | Customer | P1 |
| FR-PAY-002 | The system shall mock the payment gateway and return deterministic results. | System | P1 |
| FR-PAY-003 | The system shall debit the customer's account on successful payment. | System | P1 |
| FR-PAY-004 | The system shall record every payment with status and timestamp. | System | P1 |
| FR-PAY-005 | The system shall allow customers to view their payment history. | Customer | P1 |

### 9.6 Loan Management (FR-LOAN)

| ID | Description | Actor | Priority |
|---|---|---|---|
| FR-LOAN-001 | The system shall allow customers to submit a loan application. | Customer | P1 |
| FR-LOAN-002 | The system shall set initial loan status to PENDING. | System | P1 |
| FR-LOAN-003 | The system shall allow customers to view their own loan applications. | Customer | P1 |
| FR-LOAN-004 | The system shall allow employees/admins to approve or reject a loan. | Employee/Admin | P1 |
| FR-LOAN-005 | The system shall generate a notification on loan status change. | System | P1 |

### 9.7 Service Requests (FR-SR)

| ID | Description | Actor | Priority |
|---|---|---|---|
| FR-SR-001 | The system shall allow customers to raise a service request with category and description. | Customer | P1 |
| FR-SR-002 | The system shall set initial service request status to OPEN. | System | P1 |
| FR-SR-003 | The system shall allow customers to view their own service requests. | Customer | P1 |
| FR-SR-004 | The system shall allow employees/admins to update service request status. | Employee/Admin | P1 |
| FR-SR-005 | The system shall generate a notification on service request status change. | System | P1 |

### 9.8 Notification Management (FR-NOTIF)

| ID | Description | Actor | Priority |
|---|---|---|---|
| FR-NOTIF-001 | The system shall generate notifications for transactions, payments, loans, and service requests. | System | P1 |
| FR-NOTIF-002 | The system shall persist notifications in the database (no real email/SMS). | System | P1 |
| FR-NOTIF-003 | The system shall allow customers to view their own notification history. | Customer | P1 |

### 9.9 Fraud Monitoring (FR-FRAUD)

| ID | Description | Actor | Priority |
|---|---|---|---|
| FR-FRAUD-001 | The system shall flag transactions above a configurable threshold. | System | P1 |
| FR-FRAUD-002 | The system shall reject transactions from inactive or blocked accounts. | System | P0 |
| FR-FRAUD-003 | The system shall reject transactions exceeding available balance. | System | P0 |

### 9.10 Audit and Compliance (FR-AUDIT)

| ID | Description | Actor | Priority |
|---|---|---|---|
| FR-AUDIT-001 | The system shall log audit records for login, account creation, transfer, payment, loan actions, and service request updates. | System | P1 |
| FR-AUDIT-002 | Each audit record shall capture actor, action, timestamp, and outcome. | System | P1 |

---

## 10. Business Rules

The following business rules must be enforced by the system. They are intentionally kept simple so they can be implemented within one week.

1. **BR-001** — A customer's email address must be unique across the system.
2. **BR-002** — Passwords must never be stored in plain text. BCrypt hashing is mandatory.
3. **BR-003** — Only authenticated users can access protected banking operations.
4. **BR-004** — Customers can access only their own accounts, transactions, payments, loans, service requests, and notifications.
5. **BR-005** — Only ACTIVE accounts can perform transactions or payments.
6. **BR-006** — Transfer amount must be greater than zero.
7. **BR-007** — The source account must have sufficient balance for a transfer or payment.
8. **BR-008** — Both source and destination accounts must exist for a transfer.
9. **BR-009** — Source and destination accounts must be different.
10. **BR-010** — Every transaction must have a clear status (SUCCESS, FAILED, PENDING, FLAGGED).
11. **BR-011** — Transactions from BLOCKED or INACTIVE accounts must be rejected.
12. **BR-012** — Transactions above the configured high-value threshold must be flagged.
13. **BR-013** — Only EMPLOYEE or ADMIN roles can approve or reject loan applications.
14. **BR-014** — Only EMPLOYEE or ADMIN roles can update service request status.
15. **BR-015** — Important banking operations must generate audit records.
16. **BR-016** — Important banking operations should generate notifications.

---

## 11. Non-Functional Requirements

### 11.1 Security
- All protected APIs require a valid JWT.
- Role-based authorization is enforced at the API level.
- Passwords are hashed using BCrypt.
- Sensitive data (passwords, tokens) is never logged.
- Input validation is applied on all endpoints.

### 11.2 Performance
- APIs respond within a reasonable time for an MVP (target: < 1 second for simple reads, < 2 seconds for writes).
- No unrealistic production-grade SLAs are required.

### 11.3 Availability
- Services handle downstream failures gracefully (clear error messages, no silent failures).
- Circuit breaking is **not required** for the MVP but services should not crash on downstream errors.

### 11.4 Scalability
- Services are independently deployable.
- State is stored in databases, not in memory, so services can scale horizontally in the future.

### 11.5 Maintainability
- Clear separation of concerns: Controller, Service, Repository, DTO, Entity, Exception Handler.
- Consistent package structure across services.

### 11.6 Reliability
- Money transfers and payments maintain consistent account balances (atomic updates within a single service).

### 11.7 Observability
- Spring Boot Actuator health endpoints enabled.
- Structured logging with meaningful log levels (INFO, WARN, ERROR).

### 11.8 Auditability
- Important operations are traceable via audit records.

### 11.9 Usability (API)
- APIs return consistent, well-structured responses.
- Error responses include timestamp, status, error code, message, and path.

---

## 12. Technology Stack

### 12.1 Mandatory
- **Java 21**
- **Spring Boot 3.x**
- **Spring Web** (REST APIs)
- **Spring Data JPA** + **Hibernate**
- **Spring Security** + **JWT** + **BCrypt**
- **Spring Cloud Gateway**
- **Eureka** (Service Discovery)
- **PostgreSQL**
- **Jakarta Bean Validation**
- **JUnit 5** + **Mockito**
- **OpenAPI / Swagger**
- **Gradle**
- **Git** + **GitHub**

### 12.2 Recommended
- **RestClient** or **WebClient** for inter-service calls
- **Spring Boot Actuator** for health checks
- **Docker** + **Docker Compose** for running all services locally

### 12.3 Optional (do not introduce unless justified)
- **Redis** (caching)
- **Kafka / RabbitMQ** (event streaming)
- **Kubernetes** (orchestration)
- **AWS / cloud providers**
- **Elasticsearch** (search)

> **Guideline:** For a one-week MVP, prefer the **Mandatory** stack. Use **Recommended** items only where they add clear value. Avoid **Optional** items entirely.

---

## 13. Microservices Requirements

The system must contain **at least 6 independently deployable microservices**. The following 8 services are proposed. Services marked **Mandatory** must be implemented for the MVP. Services marked **Simplifiable** may be reduced in scope or combined if needed to meet the one-week timeline — but each must still have a clear business responsibility.

### 13.1 API Gateway Service
- **Responsibility:** Single entry point for all clients. Routes requests to downstream services. Enforces cross-cutting concerns (CORS, request logging).
- **Main capabilities:** Routing, request/response transformation, aggregation of auth headers.
- **Data owned:** None (stateless).
- **Actors:** All clients.
- **Dependencies:** Eureka, all downstream services.
- **Mandatory:** ✅ Yes

### 13.2 Authentication Service
- **Responsibility:** Handles registration, login, JWT issuance, and validation support.
- **Main capabilities:** Register, login, role assignment, password hashing.
- **Data owned:** Users, credentials, roles.
- **Actors:** Customer, Employee, Admin.
- **Dependencies:** Customer Service (to verify/create customer records during registration).
- **Mandatory:** ✅ Yes

### 13.3 Customer Service
- **Responsibility:** Manages customer profiles and onboarding.
- **Main capabilities:** Create, read, update customers; lookup by email/ID.
- **Data owned:** Customer profiles.
- **Actors:** Customer, Employee, Admin.
- **Dependencies:** None (core service).
- **Mandatory:** ✅ Yes

### 13.4 Account Service
- **Responsibility:** Manages bank accounts and balances.
- **Main capabilities:** Create account, view account, view balance, list accounts per customer, debit/credit operations.
- **Data owned:** Accounts, balances.
- **Actors:** Customer, Employee, Admin, Transaction Service, Payment Service.
- **Dependencies:** Customer Service (to validate customer existence).
- **Mandatory:** ✅ Yes

### 13.5 Transaction Service
- **Responsibility:** Processes money transfers and maintains transaction history.
- **Main capabilities:** Transfer funds, record transactions, view history, apply fraud rules.
- **Data owned:** Transactions.
- **Actors:** Customer, Employee, Admin.
- **Dependencies:** Account Service (validate, debit, credit), Notification Service (optional for MVP).
- **Mandatory:** ✅ Yes

### 13.6 Payment Service — *Simplifiable*
- **Responsibility:** Handles bill payments with a mocked payment gateway.
- **Main capabilities:** Initiate payment, record payment, view payment history.
- **Data owned:** Payments.
- **Actors:** Customer.
- **Dependencies:** Account Service (debit), Notification Service (optional).
- **Mandatory for MVP:** ⚠️ P1 — may be simplified to a single endpoint with mocked gateway.

### 13.7 Loan Service — *Simplifiable*
- **Responsibility:** Manages loan applications and approvals.
- **Main capabilities:** Submit loan, view loans, approve/reject.
- **Data owned:** Loan applications.
- **Actors:** Customer, Employee, Admin.
- **Dependencies:** Customer Service (validate customer), Notification Service (optional).
- **Mandatory for MVP:** ⚠️ P1 — may be simplified to basic CRUD + status transitions.

### 13.8 Notification Service — *Simplifiable*
- **Responsibility:** Generates and stores notifications.
- **Main capabilities:** Create notification, list notifications per customer.
- **Data owned:** Notifications.
- **Actors:** Customer, internal services.
- **Dependencies:** None.
- **Mandatory for MVP:** ⚠️ P1 — may be simplified to a synchronous REST call that persists to DB.

> **Note:** Service Requests (US-010, US-011) can be hosted inside the **Customer Service** for the MVP to avoid creating a ninth service. If the team prefers, a separate **Service Request Service** may be created, but it is not required.

---

## 14. Data and Database Requirements

- Each business service owns its own **PostgreSQL database**.
- **No service may directly access another service's database.** All cross-service data must be retrieved via REST APIs.
- Databases are named logically (e.g., `auth_db`, `customer_db`, `account_db`, `transaction_db`, `payment_db`, `loan_db`, `notification_db`).
- Detailed schemas, tables, and columns are **out of scope** for this document and will be defined in `spec.md`.
- Each service manages its own schema migrations (e.g., via Flyway or simple JPA auto-DDL for the MVP).

---

## 15. API Requirements

APIs are exposed through the API Gateway. The paths below are **conceptual** — final paths will be defined in `spec.md`.

### 15.1 Authentication
- `POST /auth/register` — Register a new customer.
- `POST /auth/login` — Authenticate and receive a JWT.

### 15.2 Customers
- `GET /customers/{id}` — View customer profile.
- `PUT /customers/{id}` — Update customer profile.
- `POST /customers` — Onboard a customer (employee/admin).

### 15.3 Accounts
- `POST /accounts` — Create a new account.
- `GET /accounts/{id}` — View account details.
- `GET /accounts/customer/{customerId}` — List accounts of a customer.
- `GET /accounts/{id}/balance` — View account balance.

### 15.4 Transactions
- `POST /transactions/transfer` — Transfer money between accounts.
- `GET /transactions/{id}` — View a specific transaction.
- `GET /transactions/account/{accountId}` — View transaction history of an account.

### 15.5 Payments
- `POST /payments` — Initiate a bill payment.
- `GET /payments/{id}` — View a specific payment.
- `GET /payments/customer/{customerId}` — View payment history.

### 15.6 Loans
- `POST /loans` — Submit a loan application.
- `GET /loans/{id}` — View a loan application.
- `GET /loans/customer/{customerId}` — View customer's loans.
- `PUT /loans/{id}/approve` — Approve a loan (employee/admin).
- `PUT /loans/{id}/reject` — Reject a loan (employee/admin).

### 15.7 Service Requests
- `POST /service-requests` — Raise a service request.
- `GET /service-requests/{id}` — View a service request.
- `GET /service-requests/customer/{customerId}` — View customer's requests.
- `PUT /service-requests/{id}/status` — Update status (employee/admin).

### 15.8 Notifications
- `GET /notifications/customer/{customerId}` — View customer's notifications.

---

## 16. Security Requirements

- **Authentication:** JWT-based. Tokens issued on login, validated on every protected request.
- **Authorization:** Role-based. Roles: `CUSTOMER`, `EMPLOYEE`, `ADMIN`.
- **Password Storage:** BCrypt hashing. Plain-text passwords are forbidden.
- **Protected APIs:** All banking operations require a valid JWT.
- **Ownership Checks:** Customers can access only their own data.
- **Input Validation:** All endpoints validate input using Jakarta Bean Validation.
- **Sensitive Data:** Passwords, tokens, and other secrets must never appear in logs or API responses.
- **Error Messages:** Login failures return generic messages (no user enumeration).

---

## 17. Inter-Service Communication

All inter-service communication is **synchronous REST** for the MVP. Example flows:

```
Transaction Service  --REST-->  Account Service    (validate, debit, credit)
Transaction Service  --REST-->  Notification Service (optional, on success)
Payment Service      --REST-->  Account Service    (debit)
Payment Service      --REST-->  Notification Service (optional)
Loan Service         --REST-->  Customer Service   (validate customer)
Loan Service         --REST-->  Notification Service (optional)
Auth Service         --REST-->  Customer Service   (create/verify customer)
```

- No event-driven architecture (Kafka/RabbitMQ) is required for the MVP.
- Timeouts and clear error handling are required on all inter-service calls.

---

## 18. Error Handling

All services return a **consistent error response** with the following conceptual fields:

- `timestamp` — when the error occurred
- `status` — HTTP status code
- `errorCode` — a stable business error code (e.g., `INSUFFICIENT_BALANCE`)
- `message` — human-readable description
- `path` — the request path

Common error scenarios:
- Customer not found
- Account not found
- Insufficient balance
- Invalid credentials
- Unauthorized access
- Invalid transaction (e.g., zero amount, same source/destination)
- Loan not found
- Payment failure
- Service request not found

Detailed error codes and response structure will be finalized in `spec.md`.

---

## 19. Logging and Auditing

### 19.1 Application Logging
- Structured logs at INFO, WARN, ERROR levels.
- No sensitive data (passwords, tokens) in logs.
- Request/response correlation via trace IDs where practical.

### 19.2 Audit Logging
Audit records are created for:
- Login (success and failure)
- Account creation
- Account updates
- Money transfers
- Payments
- Loan submission, approval, rejection
- Service request status updates

Each audit record captures:
- **WHO** — the actor (user ID and role)
- **WHAT** — the action performed
- **WHEN** — timestamp
- **OUTCOME** — success or failure (and reason on failure)

---

## 20. MVP Priorities

### 20.1 Priority Definitions
- **P0** — Mandatory for MVP. Without this, the system is not functional.
- **P1** — Important. Should be included in the MVP but may be simplified if time is tight.
- **P2** — Optional / future enhancement.

### 20.2 Minimum P0 Flow
The following flow **must** work end-to-end in the MVP:

```
Customer Registration
        ↓
Customer Login (JWT)
        ↓
View Account
        ↓
Check Balance
        ↓
Transfer Money
        ↓
View Transaction History
```

### 20.3 P1 Functionality
These should be included if time permits, possibly in simplified form:
- Bill payments
- Loan applications and approvals
- Service requests
- Notifications
- Fraud flagging
- Audit logging
- Employee onboarding

### 20.4 P2 Functionality (Future)
- Real payment gateway integration
- Real email/SMS notifications
- Advanced fraud detection
- Multi-currency support
- UI layer

---

## 21. Assumptions and Constraints

### 21.1 Assumptions
- The development team has basic familiarity with Java, Spring Boot, and PostgreSQL.
- A local development environment with Java 21, Gradle, and PostgreSQL is available.
- Docker is available but not mandatory for the MVP.
- External systems (payment gateway, email/SMS) are mocked.
- The MVP is educational — it does not need to handle production banking loads.

### 21.2 Constraints
- **Timeline:** ~1 week for a small team.
- **Scope:** Simplicity over feature volume.
- **No real money movement.**
- **No real external integrations.**
- **No advanced infrastructure** (Kubernetes, Kafka, cloud providers).

---

## 22. Future Enhancements

The following are explicitly deferred to post-MVP iterations:

- Real payment gateway integration
- Real email and SMS notifications
- Event-driven architecture (Kafka/RabbitMQ) for notifications and audit
- Machine learning-based fraud detection
- Credit bureau and KYC/AML integrations
- Multi-currency and international transfers
- Mobile and web UI
- Kubernetes deployment
- Advanced observability (distributed tracing, metrics dashboards)
- Production-grade disaster recovery

---

## 23. Traceability Matrix

The matrix below connects each user story to its functional requirements, responsible microservice, and priority.

| User Story | Functional Requirement(s) | Microservice | Priority |
|---|---|---|---|
| US-001 — Customer Registration | FR-AUTH-001, FR-AUTH-002, FR-AUTH-003 | Auth Service, Customer Service | P0 |
| US-002 — Customer Login | FR-AUTH-004, FR-AUTH-005, FR-AUTH-006 | Auth Service | P0 |
| US-003 — Employee/Admin Login | FR-AUTH-004, FR-AUTH-005, FR-AUTH-006 | Auth Service | P0 |
| US-004 — View Account Balance | FR-ACC-003, FR-ACC-005 | Account Service | P0 |
| US-005 — Transfer Money | FR-TXN-001, FR-TXN-002, FR-TXN-003, FR-TXN-004, FR-TXN-005 | Transaction Service, Account Service | P0 |
| US-006 — View Transaction History | FR-TXN-006 | Transaction Service | P0 |
| US-007 — Pay a Bill | FR-PAY-001, FR-PAY-002, FR-PAY-003, FR-PAY-004 | Payment Service, Account Service | P1 |
| US-008 — Apply for a Loan | FR-LOAN-001, FR-LOAN-002 | Loan Service | P1 |
| US-009 — Approve/Reject Loan | FR-LOAN-004, FR-LOAN-005 | Loan Service | P1 |
| US-010 — Raise Service Request | FR-SR-001, FR-SR-002 | Customer Service (or Service Request Service) | P1 |
| US-011 — Update Service Request | FR-SR-004, FR-SR-005 | Customer Service (or Service Request Service) | P1 |
| US-012 — View Notifications | FR-NOTIF-003 | Notification Service | P1 |
| US-013 — View Customer Profile | FR-CUST-001 | Customer Service | P1 |
| US-014 — View Account Details | FR-ACC-002, FR-ACC-004 | Account Service | P0 |
| US-015 — High-Value Transaction Flagging | FR-FRAUD-001 | Transaction Service | P1 |
| US-016 — Audit Logging | FR-AUDIT-001, FR-AUDIT-002 | All services (audit module) | P1 |
| US-017 — Employee Onboards Customer | FR-CUST-003, FR-CUST-005 | Customer Service | P1 |
| US-018 — View Payment History | FR-PAY-005 | Payment Service | P1 |

---

## 24. Quality Checklist (Self-Verification)

Before development begins, the team should confirm the following:

- ✅ At least 15 user stories defined (18 provided).
- ✅ At least 6 microservices defined (8 proposed; 5 mandatory + 3 simplifiable).
- ✅ Every microservice has a clear responsibility and data ownership.
- ✅ Requirements are testable and unambiguous.
- ✅ No implementation code, SQL scripts, or configuration files included.
- ✅ Project is realistically achievable in one week.
- ✅ Security, authentication, and authorization requirements are present.
- ✅ Database-per-service approach is defined.
- ✅ Inter-service REST communication is documented.
- ✅ API Gateway and Eureka service discovery are addressed.
- ✅ PostgreSQL is the chosen database.
- ✅ REST APIs are defined at a conceptual level.
- ✅ Audit logging and basic fraud monitoring are addressed.
- ✅ Notifications, loans, payments, and service requests are covered.
- ✅ Customer and employee workflows are included.
- ✅ P0 / P1 / P2 priorities are clearly assigned.
- ✅ Every user story has acceptance criteria.
- ✅ A traceability matrix is provided.
- ✅ No unnecessary technologies have been introduced.

---

**End of Requirement Specification — InfyBank360 v1.0**

> The next step is to produce the technical specification (`spec.md`), which will describe **how** each requirement is implemented — including package structure, entity design, API contracts, database schemas, and service configurations.