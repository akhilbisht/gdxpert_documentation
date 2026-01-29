# GDXpert with Blockbrain

## Document Heading Hierarchy

```
1. Introduction and Goals
   1.1 Architecture Overview
   1.2 Quality Goals
2. Component Description
   2.1 Blockbrain
   2.2 GDXpert
   2.3 API Layer
3. Authentication & Access Model
   3.1 User Authentication (GDXpert)
   3.2 Separation of Tenants
   3.3 GDXpert → Blockbrain Authorization Model
   3.4 Blockbrain OAuth Authentication (Per Request / Session)
   3.5 Blockbrain Authorization Decision
       3.5.1 OAuth Token Validation
       3.5.2 Service Identity Verification
       3.5.3 API Key Validation
   3.6 Request Flow
       3.6.1 Flow A: Publishing / Management Flow
       3.6.2 Flow B: Chatting Flow
4. Data Flow & Storage
   4.1 Data Request Flow
   4.2 Data Exchange Between Blockbrain & GDXpert
   4.3 Data Storage
5. Security Measures
   5.1 Encryption
   5.2 Access Controls on GDXpert
   5.3 Logging & Monitoring
6. Summary
```

---

## 1. Introduction and Goals

This document outlines the end-to-end architecture, authentication model, data flow, security controls, potential vulnerabilities, and common operational scenarios for the Blockbrain–GDXpert platform, with a focus on clarity for both technical and security stakeholders.

### 1.1 Architecture Overview

The platform is composed of three primary components:

- **GDXpert** – Domain-specific expert system and execution layer
- **Blockbrain** – Core intelligence and orchestration layer
- **API Layer** – Secure integration and access interface by which Blockbrain is accessed inside GDXpert

Each component is loosely coupled, communicates over secure APIs, and follows a defense-in-depth security approach.

### 1.2 Quality Goals

*[Quality goals to be defined]*

---

## 2. Component Description

The integration links G+D Xpert with the Blockbrain AI platform so that users can invoke advanced search and analytics directly from the existing web component without altering current client workflows. G+D Xpert sends requests through an API gateway that authenticates via today's API-key (and future OAuth 2.0 client-credentials) while enforcing strict IP-whitelisting, rate limiting and logging.

### Component Overview

*[Component diagram reference]*

### 2.1 Blockbrain

#### Role

Blockbrain functions as a Retrieval-Augmented Generation (RAG) system and serves as the central intelligence and orchestration layer of the platform. It combines information retrieval with generative reasoning to produce accurate, context-aware responses.

#### Key Responsibilities of Blockbrain

- Interpreting user intent and queries
- Retrieving relevant knowledge from approved data sources (vector stores, document repositories)
- Augmenting prompts with retrieved context
- Applying business rules and reasoning logic
- Orchestrating downstream workflows, including calls to GDXpert
- Aggregating, validating, and post-processing responses

#### RAG-Specific Capabilities

- Document ingestion and embedding generation
- Vector similarity search for contextual retrieval
- Context filtering and relevance scoring
- Prompt construction using retrieved knowledge
- Response grounding to reduce hallucinations

#### Characteristics

- Stateless at runtime (context passed via tokens and request payloads)
- Horizontally scalable to handle variable query loads
- No direct user authentication handling (delegated to Identity Provider)
- Supports multiple knowledge sources with access-controlled retrieval

### 2.2 GDXpert

#### Role

GDXpert acts as an access-aware integration and execution system that works in close coordination with Blockbrain. Its primary function is to ensure that each user interacts with the system strictly within their authorized access level, particularly with respect to document visibility and domain-specific operations.

GDXpert does not directly expose data to users; instead, it enforces access constraints while enabling Blockbrain to deliver personalized, permission-safe responses.

#### Key Responsibilities of GDXpert

- Enforcing document- and domain-level access controls
- Validating user access context received from Blockbrain
- Executing domain-specific logic only on authorized data
- Filtering responses so users receive only permitted information
- Supporting multi-tenant and role-based segregation of data

#### Access-Control Capabilities

- Document-level authorization (ACLs, roles, or attributes)
- Integration with enterprise identity claims (roles, groups, scopes)
- Context-aware execution based on user access level

#### Interaction with Blockbrain (RAG Context)

- Receives user access metadata (role, group)
- Restricts retrieval or processing to documents aligned with the user's access
- Returns only access-approved results to Blockbrain
- Supports fine-grained filtering to ensure RAG responses are grounded in authorized content

#### Characteristics

- No direct user authentication or UI exposure
- Accessible only via secure internal APIs
- Designed for least-privilege execution
- Scalable to support multiple users with varying access levels concurrently

### 2.3 API Layer

#### Role

The API layer is the secure gateway that exposes system functionality to users and other systems.

#### Key Responsibilities

- Authentication and authorization enforcement
- Rate limiting and throttling
- Request validation and schema enforcement
- Secure routing to Blockbrain services

#### Characteristics

- Acts as a single entry point
- Integrates with Identity Providers (IdP)
- Supports both internal and external consumers

---

## 3. Authentication & Access Model

This section describes the end-to-end authentication and access flow between GDXpert and Blockbrain, highlighting strict tenant separation, service-level authorization, and controlled user access.

*Figure 1: User Authentication Diagram*

### Roles & Responsibilities

- Creation of API: Who has the access? Is this a service account or an individual?
- API is not bounded to specific users
- Replace API key with Service User
- **Status**: We will have a service user with read-only access that will be used for the authentication & authorization between GDX & BB (In dev we have API Keys)
- Connection to BB is read-only, BB is not managing any user information/access. Entirely managed by GDXpert
- User access metadata managed in GDX

### Data Stored

- Original Data is stored in GDS
- Indexing & chunking of the data is present in BB
- Inline reference – Open the OG doc GDS, only with access can view them

*Overall Authentication flow Diagram (provided by vendor)*

### 3.1 User Authentication (GDXpert)

#### User Context

- The user is an employee of GDXpert
- The user is authenticated using GDXpert Entra ID, which belongs to a separate Entra ID tenant from Blockbrain

#### Authentication Method

- Authentication is performed via GDXpert Entra ID
- Access is granted based on Entra ID group membership

#### Outcome

- Successful authentication grants access only to GDXpert
- Authentication in GDXpert does not automatically grant access to Blockbrain

*Authentication Mechanism (Provided by Vendor)*

### 3.2 Separation of Tenants

- GDXpert and Blockbrain operate in different Entra ID tenants
- Blockbrain has its own OAuth authority, client registrations, and access controls
- User identities authenticated in GDXpert are not trusted or reused directly by Blockbrain
- This structure enforces strong isolation and prevents identity or permission leakage across tenants

### 3.3 GDXpert → Blockbrain Authorization Model

#### Service-Level Access

- GDXpert connects to Blockbrain using one shared service API key
- This API key is common for all users accessing Blockbrain through GDXpert
- End users never authenticate directly against Blockbrain

#### Implication

- Blockbrain authorizes GDXpert as a service, not individual users
- User-level access enforcement is handled logically via request context and metadata

### 3.4 Blockbrain OAuth Authentication (Per Request / Session)

Every request or session initiated by GDXpert towards Blockbrain is authenticated using Blockbrain's OAuth service.

#### OAuth Flow Details

- Authentication uses Client ID and Client Secret registered in Blockbrain's tenant
- A short-lived OAuth access token is generated
- Default token lifetime: 1 hour (configurable)

#### Credential Handling

- Client ID and Client Secret are securely stored on the GDXpert server
- OAuth token is only requested by GDXpert Server and forwarded to Blockbrain requests
- OAuth access tokens are:
  - Stored only in memory (RAM)
  - Never written to disk
  - Never shared via file systems or logs

### 3.5 Blockbrain Authorization Decision

For each OAuth-authenticated request, Blockbrain performs a multi-step authorization evaluation before allowing any interaction.

#### How the Authorization Decision Is Made

Blockbrain evaluates authorization through a sequential, layered validation process. Each step must succeed for the request to proceed. Failure at any stage results in immediate rejection.

#### 3.5.1 OAuth Token Validation

- Validates the token signature against the Blockbrain OAuth authority
- Checks token expiry
- Verifies the token issuer
- Ensures the token was issued for the correct Client ID (audience check)

#### 3.5.2 Service Identity Verification

- Confirms the calling service corresponds to a registered GDXpert client
- Verifies the Client ID is active and not revoked

#### 3.5.3 API Key Validation

- Validates the API key included in the request
- Confirms the key is active and not revoked
- Maps the key to a registered application/service context

### 3.6 Request Flow

#### 3.6.1 Flow A: Publishing / Management Flow

**Purpose**: Used for non-interactive system operations such as document ingestion, publishing, or metadata updates.

**Request Path**: GDXpert → Blockbrain API

**Headers**:
- OAuth Access Token
- Current API Key
- Request Type

**Request Type**:
- GET / POST / DELETE

**Target**:
- Database-Id

**Processing in Blockbrain**:
- Request is validated and authenticated
- Authorization policies are rechecked for publishing operations
- Payload is validated against publishing schemas
- Data is processed, stored, or indexed as required (e.g., RAG ingestion)
- Success or failure response is returned to GDXpert

#### 3.6.2 Flow B: Chatting Flow

**Purpose**: Used for interactive, user-driven communication such as RAG-based question answering or conversational queries.

**Client Layer**: Chat

##### Step 1: Client → Proxy Service

**Headers**:
- easybrowse AccessToken
- Org-Id
- Request Type (read only)
- GET / POST / DELETE

**Target**:
- WebComponent-UID

##### Step 2: Proxy Service → Blockbrain

**Proxy Responsibilities**:
- Validates client session and access token
- Injects server-side credentials

**Headers Added by Proxy**:
- OAuth Access Token (Blockbrain OAuth)
- Org-Id
- Current API Key

**Processing in Blockbrain**:
- Proxy-authenticated request is received
- OAuth token and API key are validated
- Org-Id is used to enforce data and document isolation
- RAG pipeline executes using only authorized content
- Response is generated and returned via the proxy

**This layered flow ensures**:
- End users never see Blockbrain credentials
- Service-level trust is preserved
- Fine-grained, organization-aware RAG responses

*Flow Diagrams (Provided by vendor)*

---

## 4. Data Flow & Storage

Blockbrain may contain derived or indexed representations of GDXpert data, but it is not the authoritative source. Specifically:

### Blockbrain can store:

- Indexed documents or content snapshots
- Vector embeddings used for RAG retrieval
- Metadata and summaries synchronized from GDXpert

### Blockbrain does not store:

- Raw source-of-truth records
- Sensitive transactional data
- Domain execution logic

### Key Principle

- GDXpert is the system of record for domain data and logic
- Blockbrain is an intelligent retrieval and orchestration layer, optimized for fast, context-aware responses

### This separation ensures:

- Strong data governance
- Reduced duplication of sensitive data
- Better performance through cached and indexed knowledge
- Correctness by deferring authoritative decisions to GDXpert when required

### 4.1 Data Request Flow

1. User sends request to API layer
2. API authenticates and authorizes the request
3. Request is forwarded to Blockbrain
4. Blockbrain processes intent and determines required actions
5. Blockbrain invokes GDXpert via internal API
6. GDXpert executes domain logic
7. Response flows back to Blockbrain
8. Final response returned to the user via API

### 4.2 Data Exchange Between Blockbrain & GDXpert

Blockbrain and GDXpert exchange data through secure, internal APIs to enable coordinated request processing while maintaining clear system boundaries. Blockbrain acts as the orchestration and intelligence layer, while GDXpert remains the authoritative domain execution system. Data exchanged between the two systems is purpose-driven, access-controlled, and limited to what is strictly necessary to fulfill each request.

#### Data Types

- Structured request payloads (JSON)
- Contextual metadata (request ID, user role, correlation ID)
- Execution results and status codes

#### Constraints

- No PII passed unless strictly required
- Data minimized to the least required fields

### 4.3 Data Storage

Blockbrain and GDXpert follow a minimal, security-first data storage strategy. Data is stored only when necessary to support system performance, reliability, auditing, and operational needs. Sensitive data exposure is avoided by design, and all stored data adheres to strict lifecycle and access controls.

#### Transient Data (In-Memory / Short-Lived Caches)

Transient data is used to optimize performance and reduce latency. This includes:

- OAuth access tokens and session context stored only in memory (RAM)
- Short-lived request context such as correlation IDs and authorization decisions
- Cached RAG retrieval results or embeddings to improve response times

**Transient data**:
- Has defined time-to-live (TTL)
- Is automatically purged on expiry or service restart
- Is never written to disk or shared storage
- Is inaccessible to unauthorized services or users

#### Persistent Data

Persistent storage is limited to operationally necessary information:

- Audit logs capturing authentication attempts, authorization decisions, request metadata, and system events for compliance and security monitoring
- Configuration metadata such as service settings, feature flags, and access mappings
- Execution summaries (if required) including non-sensitive request outcomes, timestamps, and status indicators for troubleshooting and analytics

#### Credential & Secret Handling

- No raw credentials, secrets, or user passwords are stored in Blockbrain or GDXpert
- API keys, client secrets, and certificates are stored only in secure secret management systems (e.g., vault services) *[NEED MORE CLARITY]*
- Runtime access to secrets is controlled, audited, and time-bound

This approach ensures strong data governance, minimizes risk exposure, and aligns with enterprise security and compliance standards.

---

## 5. Security Measures

The platform is designed with a defense-in-depth security approach to protect data, services, and access at every layer. Security controls are applied consistently across network communication, identity and access management, data handling, and operational monitoring to ensure confidentiality, integrity, and availability.

### 5.1 Encryption

Strong encryption mechanisms are used to safeguard data both in transit and at rest, ensuring that sensitive information remains protected from unauthorized access or interception throughout its lifecycle.

- TLS 1.2+ for all data in transit
- Encryption at rest using platform-managed keys
- Secrets stored in a secure vault (e.g., Key Vault) *[NEED MORE CLARITY]*

### 5.2 Access Controls on GDXpert

GDXpert enforces strict access control mechanisms to ensure that users and services can only access resources aligned with their assigned roles and permissions. Access is governed at multiple levels—identity, scope, and network—to minimize unauthorized access and reduce the overall attack surface.

- RBAC for internal and external users
- Scope-based access for external users
- Network-level restrictions (private endpoints, firewalls)
- Mutual TLS for internal service-to-service communication (optional)

### 5.3 Logging & Monitoring

Blockbrain and GDXpert implement centralized logging and monitoring to ensure full visibility across authentication, authorization, and request processing flows. Logs are designed to support operational monitoring, troubleshooting, and security auditing, while enabling end-to-end traceability of user and system interactions.

- Centralized logging for API, Blockbrain, and GDXpert
- Correlation IDs for end-to-end traceability
- Security logs for authentication and authorization events

---

## 6. Summary

This architecture ensures:

### Clear separation of responsibilities

This is achieved by clearly delineating system roles: GDXpert handles user authentication, access context, and domain-specific logic, while Blockbrain functions as an independent RAG and orchestration layer responsible for intent understanding, policy enforcement, and response generation. The API layer acts as a controlled entry point, ensuring that no single component assumes multiple responsibilities.

### Secure and scalable authentication for internal and external users

Authentication is layered and tenant-isolated. Internal users authenticate through GDXpert Entra ID, while Blockbrain relies on OAuth-based service authentication using client credentials. This service-to-service model avoids direct end-user authentication against Blockbrain, enabling scalability, reducing identity sprawl, and maintaining strict tenant boundaries.

### Controlled and minimal data exchange

Only strictly required, non-sensitive data is exchanged between GDXpert and Blockbrain using structured, schema-validated payloads. Contextual metadata is passed to support request traceability and authorization, while unnecessary attributes—especially PII—are excluded by design. This minimizes data exposure and reduces compliance risk.

### Predictable behavior under failure conditions

The system is designed to fail fast and safely. Authentication or authorization failures result in immediate request rejection with clear error codes, while downstream services are never invoked. Timeouts, retries, and circuit-breaking patterns ensure that partial failures do not cascade, providing consistent and predictable system behavior.

---

## GDXpert UI

*[UI documentation to be added]*

---

*Document End*
