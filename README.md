# DocFlow — Employee Management System (EMS)
### Formal Technical Specification & Software Requirements Document

---

## 1. Executive Summary & System Overview

**DocFlow** is an enterprise-grade Employee Management System (EMS) backend engineered with **FastAPI**, **MySQL**, and **Google OAuth 2.0 / OpenID Connect (OIDC)**. The system serves as the central administrative directory for managing employee identity lifecycles, role permissions, and organizational metadata while delegating authentication to Google's identity provider.

### Core Objectives
- **Centralized Identity Management**: Single Sign-On (SSO) integration using Google OAuth 2.0.
- **Automated Provisioning**: JIT (Just-In-Time) provisioning of employee records upon first identity verification.
- **High-Throughput ASGI Backend**: Non-blocking asynchronous REST endpoints powered by Starlette and Uvicorn.
- **Resilient Relational Storage**: MySQL persistence managed via SQLAlchemy 2.0 with connection pooling and health telemetry.

---

## 2. Technology Stack

| Layer | Component | Version / Specification | Rationale |
|---|---|---|---|
| **Language Runtime** | Python | `>= 3.10` (tested on 3.14) | High productivity, native type hinting, robust ecosystem. |
| **Web Framework** | FastAPI | `^0.115.0` | High performance, automatic OpenAPI/Swagger documentation, async ASGI support. |
| **ASGI Server** | Uvicorn | `^0.30.0` | Production-ready, lightning-fast ASGI implementation based on `uvloop` & `httptools`. |
| **Database Engine** | MySQL | `8.0+` | ACID compliance, robust indexing (B-Tree), relational integrity for employee records. |
| **ORM & Dialect** | SQLAlchemy + PyMySQL | `2.0+` / `1.1+` | Modern declarative ORM, connection pooling (`pool_pre_ping`), and parameter-binding security. |
| **Authentication** | Authlib + Starlette Session | `^1.3.0` | Compliant OpenID Connect (OIDC) client with automated CSRF state validation. |
| **Environment Mgmt** | python-dotenv | `^1.0.0` | Strict Twelve-Factor App compliance for runtime configuration. |

---

## 3. System Architecture & Information Flow

### High-Level Architecture

```mermaid
graph TD
    Client[Web Client / Frontend Application] -->|HTTP / REST API| FastAPI[FastAPI Gateway - main.py]
    FastAPI -->|Session Middleware| StarletteSession[Cookie Session Manager]
    FastAPI -->|OAuth 2.0 / OIDC Flow| GoogleIdP[Google Identity Provider]
    FastAPI -->|SQLAlchemy Engine & Pool| MySQL[(MySQL Database - docflow_db)]
    
    subgraph Core Endpoints
        FastAPI --> AuthRoute["/login & /auth/callback"]
        FastAPI --> UserRoute["/me & /employees"]
        FastAPI --> HealthRoute["/health & /db-check"]
    end
```

### Google OAuth 2.0 Authentication Sequence

```mermaid
sequenceDiagram
    autonumber
    actor User as Employee / User
    participant Browser as Client Browser
    participant API as FastAPI Backend (main.py)
    participant Google as Google Identity Provider (IdP)
    participant DB as MySQL Database

    User->>Browser: Click "Sign in with Google"
    Browser->>API: GET /login
    API->>API: Generate cryptographic state & redirect URI
    API-->>Browser: 302 Redirect to Google Consent Screen
    Browser->>Google: Authenticate & Authorize Scopes (email, profile)
    Google-->>Browser: 302 Redirect to /auth/callback?code=...&state=...
    Browser->>API: GET /auth/callback?code=...
    API->>Google: Exchange Authorization Code for Access & ID Tokens
    Google-->>API: Return Tokens & User Claims (sub, email, name, picture)
    API->>DB: Query Employee by email / google_id
    alt Employee Not Found
        API->>DB: INSERT INTO employees (JIT Provisioning)
    else Employee Exists
        API->>DB: UPDATE employees (refresh metadata)
    end
    API->>Browser: Set encrypted session cookie (docflow_session)
    API-->>Browser: 200 OK {"message": "Login successful", "user": {...}}
```

---

## 4. Software Requirements Specification (SRS)

### 4.1 Functional Requirements (FR)

| ID | Module | Requirement Description | Priority |
|---|---|---|---|
| **FR-01** | Authentication | The system must allow users to authenticate using Google OAuth 2.0 with `openid`, `email`, and `profile` scopes. | Critical |
| **FR-02** | Provisioning | The system must automatically provision a new `Employee` record if the authenticated Google email does not exist in MySQL (Just-In-Time Provisioning). | High |
| **FR-03** | Profile Sync | The system must synchronize and update employee name, avatar URL (`picture`), and Google subject ID (`google_id`) on each successful login. | Medium |
| **FR-04** | Session Auth | The system must issue and validate encrypted session cookies to maintain stateful authorization across subsequent requests (`/me`). | High |
| **FR-05** | Directory Access | The system must provide an endpoint (`GET /employees`) returning a collection of active employees with their departmental and role designations. | High |
| **FR-06** | Session Invalidation | The system must provide a logout mechanism (`POST /logout`) that clears active session cookies immediately. | Medium |
| **FR-07** | Database Diagnostic | The system must provide a `/db-check` endpoint executing `SELECT 1` to verify alive MySQL connection pool status. | Low |

### 4.2 Non-Functional Requirements (NFR)

#### Security & Compliance
- **NFR-S1 (Zero Secret Exposure)**: Sensitive keys (`GOOGLE_CLIENT_SECRET`, `SECRET_KEY`, database passwords) must never be checked into version control. Strict `.gitignore` enforcement.
- **NFR-S2 (CSRF Mitigation)**: OAuth 2.0 authorization requests must include a cryptographically random `state` parameter verified upon callback receipt.
- **NFR-S3 (Injection Prevention)**: All SQL transactions must utilize SQLAlchemy parameterized queries to prevent SQL Injection vulnerabilities.

#### Performance & Reliability
- **NFR-P1 (Latency)**: API response time for cached/session queries (`/me`) must not exceed 50ms under normal load.
- **NFR-P2 (Connection Pooling)**: The database engine must configure `pool_pre_ping=True` to detect and discard stale or closed socket connections transparently.
- **NFR-P3 (Fault Isolation)**: Inability to reach MySQL during application boot must log critical alerts without abruptly crashing the ASGI loop, enabling diagnostics via `/health`.

---

## 5. Database Schema & Data Dictionary

### Entity-Relationship Diagram (ERD)

```mermaid
erDiagram
    EMPLOYEES {
        int id PK "Auto-increment primary key"
        varchar google_id UK "Google Sub identifier (unique)"
        varchar email UK "Corporate email address (unique)"
        varchar name "Full name of employee"
        varchar picture "URL to Google profile photo"
        varchar role "Role designation (admin, manager, employee)"
        varchar department "Assigned business unit / department"
    }
```

### Table Specification: `employees`

| Field | Type | Modifiers | Description |
|---|---|---|---|
| `id` | `INT` | `PRIMARY KEY`, `AUTO_INCREMENT`, `NOT NULL` | Unique internal identifier for the employee record. |
| `google_id` | `VARCHAR(255)` | `UNIQUE`, `INDEX`, `NULLABLE` | Unique subject identifier (`sub`) provided by Google OpenID Connect. |
| `email` | `VARCHAR(255)` | `UNIQUE`, `INDEX`, `NOT NULL` | Primary corporate email address used for identity mapping. |
| `name` | `VARCHAR(255)` | `NULLABLE` | Full display name synchronized from Google userinfo. |
| `picture` | `VARCHAR(500)` | `NULLABLE` | HTTPS CDN URI to employee's Google account avatar. |
| `role` | `VARCHAR(50)` | `DEFAULT 'employee'`, `NOT NULL` | Access tier: `admin`, `manager`, or `employee`. |
| `department` | `VARCHAR(100)` | `NULLABLE` | Department designation (e.g., Engineering, Sales, HR). |

### SQL DDL Definition

```sql
CREATE TABLE IF NOT EXISTS employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    google_id VARCHAR(255) UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255),
    picture VARCHAR(500),
    role VARCHAR(50) DEFAULT 'employee' NOT NULL,
    department VARCHAR(100),
    INDEX idx_employee_email (email),
    INDEX idx_employee_google_id (google_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 6. REST API Specification

### Endpoints Matrix

| HTTP Method | Route | Description | Access Level | Request Body | Response Codes |
|---|---|---|---|---|---|
| `GET` | `/` | Service health & welcome status | Public | None | `200` |
| `GET` | `/db-check` | MySQL connection ping verification | Public | None | `200`, `503` |
| `GET` | `/login` | Initiates Google OAuth 2.0 redirect | Public | None | `302`, `400` |
| `GET` | `/auth/callback` | OAuth redirect consumer & token exchange | Public | Query params | `200`, `400` |
| `GET` | `/me` | Retrieves active user profile from session | Authenticated | None | `200`, `401`, `404` |
| `GET` | `/employees` | Lists all registered employees | Public / Internal | None | `200` |
| `POST` | `/logout` | Invalidates active session cookie | Authenticated | None | `200` |

### Sample Payloads

#### `GET /me` (Success Response - 200 OK)
```json
{
  "id": 1,
  "name": "Jane Doe",
  "email": "jane.doe@company.com",
  "role": "employee",
  "department": "Engineering",
  "picture": "https://lh3.googleusercontent.com/a/ACg8oc..."
}
```

#### `GET /employees` (Success Response - 200 OK)
```json
[
  {
    "id": 1,
    "name": "Jane Doe",
    "email": "jane.doe@company.com",
    "role": "admin",
    "department": "Engineering"
  },
  {
    "id": 2,
    "name": "John Smith",
    "email": "john.smith@company.com",
    "role": "employee",
    "department": "Finance"
  }
]
```

---

## 7. Configuration & Environment Variables

All runtime configuration parameters are declared in `.env` and consumed via `python-dotenv`:

| Variable | Type | Default Value | Description |
|---|---|---|---|
| `DATABASE_URL` | String | `mysql+pymysql://docflow:DocFlow123!@localhost:3306/docflow_db` | SQLAlchemy MySQL connection string. |
| `GOOGLE_CLIENT_ID` | String | *Required from GCP* | Google Cloud OAuth 2.0 Web Client ID. |
| `GOOGLE_CLIENT_SECRET` | String | *Required from GCP* | Google Cloud OAuth 2.0 Client Secret. |
| `GOOGLE_REDIRECT_URI` | String | `http://localhost:8000/auth/callback` | Authorized OAuth 2.0 callback URL. |
| `SECRET_KEY` | String | `docflow-secret-key-12345` | Cryptographic secret used to sign session cookies. |

---

## 8. Installation, Local Setup & Verification

### 8.1 Prerequisites
- Python 3.10 or higher
- MySQL Server 8.0+ running locally or in Docker
- A Google Cloud Platform (GCP) project with OAuth Credentials configured

### 8.2 Setup Steps

```bash
# 1. Clone repository
git clone https://github.com/jiyajahnavi/docflow.git
cd docflow

# 2. Activate virtual environment
python3 -m venv .venv
source .venv/bin/activate

# 3. Install dependencies
pip install -e .

# 4. Configure environment
cp .env.example .env
# Edit .env with your Google Client ID, Secret, and MySQL credentials
```

### 8.3 Running the Application

```bash
# Option A: Run via uvicorn directly with hot reload
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# Option B: Run via root main.py runner
python main.py
```

### 8.4 Verification
- **OpenAPI Documentation**: [http://localhost:8000/docs](http://localhost:8000/docs)
- **ReDoc Documentation**: [http://localhost:8000/redoc](http://localhost:8000/redoc)
- **Database Diagnostic**: [http://localhost:8000/db-check](http://localhost:8000/db-check)
