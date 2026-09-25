# ACME Salary Management System - Master Architecture Specification

Welcome to the master architectural documentation for the **ACME Salary Management System**. This comprehensive guide details the design, end-to-end data flows, backend services, frontend user interface, security model, data structures, and operational workflows of the platform.

---

## 1. Executive Summary & Plain-English Overview

> 💡 **For Non-Technical Stakeholders & HR Executives**:
> 
> Imagine a **global corporation** with 10,000 employees working across 10 countries (USA, Canada, UK, Germany, France, India, Australia, Japan, Singapore, Brazil). Managing salaries, bonuses, tax deductions, and regional payroll budgets across multiple currencies and departments manually is nearly impossible.
> 
> The **ACME Salary Management System** acts as a **smart digital command center**:
> 
> 1. **The Front Desk (Frontend UI)**: A sleek, interactive web console where HR managers can search for any employee in milliseconds, update salary structures, view interactive world maps showing active headcount per country, and explore visual financial charts.
> 2. **The Processing Engine (Backend API)**: A high-speed administrative server built with Java that verifies every request, enforces company payroll rules, and ensures numbers balance perfectly.
> 3. **The Central Vault (Database)**: A secure, structured database storing all employee contracts, department records, and compensation metrics.
> 4. **The Security Guard (Spring Security)**: An automated security barrier ensuring only authenticated HR administrators with valid credentials (`hr_admin`) can access confidential salary data.

---

## 2. End-to-End System Architecture

The platform follows a modern, decoupled **Client-Server Architecture** communicating over secure JSON REST APIs.

```mermaid
graph TB
    subgraph Client Layer [Frontend - Angular 17 SPA]
        UI_Home[Home View - Leaflet World Map]
        UI_Dir[Employee Directory - Server Table]
        UI_Analytics[Analytics Dashboard - Chart.js]
        Auth_Int[BasicAuth HTTP Interceptor]
    end

    subgraph Security Layer [Spring Security 6 Gatekeeper]
        Sec_Filter[HTTP Basic Auth & CORS Filter]
    end

    subgraph Service Layer [Backend - Spring Boot 3 REST API]
        Ctrl_Emp[EmployeeController]
        Ctrl_Ana[AnalyticsController]
        Ctrl_Seed[SeederController]
        
        Svc_Emp[EmployeeServiceImpl]
        Svc_Ana[AnalyticsServiceImpl]
        Svc_Seed[SeederServiceImpl]
        
        Repo_Emp[EmployeeRepository]
        Repo_Comp[CompensationRepository]
    end

    subgraph Persistence Layer [Database]
        DB[(MySQL 8 Database: acme_salary_db)]
    end

    UI_Home --> Auth_Int
    UI_Dir --> Auth_Int
    UI_Analytics --> Auth_Int

    Auth_Int -- HTTP REST (Basic Auth Header) --> Sec_Filter
    
    Sec_Filter -- Authorized Request --> Ctrl_Emp
    Sec_Filter -- Authorized Request --> Ctrl_Ana
    Sec_Filter -- Authorized Request --> Ctrl_Seed

    Ctrl_Emp --> Svc_Emp
    Ctrl_Ana --> Svc_Ana
    Ctrl_Seed --> Svc_Seed

    Svc_Emp --> Repo_Emp
    Svc_Emp --> Repo_Comp
    Svc_Ana --> Repo_Comp
    Svc_Seed --> Repo_Emp

    Repo_Emp -- JPQL / SQL Queries --> DB
    Repo_Comp -- JPQL / SQL Queries --> DB
```

### Full Technology Stack Table

| Layer | Component / Technology | Specification & Role |
| :--- | :--- | :--- |
| **Frontend Framework** | Angular 17.3 | Component-driven Single Page Application using Standalone Components. |
| **Frontend State & Reactive** | RxJS 7.8 | Reactive data streams with debouncing (`debounceTime`), request cancellation (`switchMap`), and event broadcasting (`BehaviorSubject`). |
| **Mapping Engine** | Leaflet.js 1.9 | GIS World Map rendering CartoDB Dark Voyager basemap tiles with custom callout badges. |
| **Charting Engine** | Chart.js 4 & ng2-charts 5 | Interactive departmental compensation bar charts, country distribution pie charts, and salary histograms. |
| **UI Design System** | TailwindCSS 3 & PrimeNG Lara Dark | Custom dark-mode aesthetic with glassmorphism overlays and responsive grid layouts. |
| **Backend Framework** | Spring Boot 3.2.5 (Java 17) | Enterprise REST API server built on Java 17 features (records, text blocks, pattern matching). |
| **Security Layer** | Spring Security 6 | HTTP Basic Authentication, CORS policies, and role-based access control (`ROLE_HR`). |
| **ORM & Persistence** | Spring Data JPA & Hibernate 6 | Relational data mapping with JPA criteria queries, custom JPQL aggregations, and entity graphs. |
| **Database Engine** | MySQL 8.0 | Relational storage database running on `acme_salary_db` with index optimizations. |

---

## 3. End-to-End User Journeys & Request Flows

### User Journey A: HR Manager Searches & Updates Employee Salary

```mermaid
sequenceDiagram
    autonumber
    actor HR as HR Manager
    participant UI as Angular Employee Directory
    participant Interceptor as BasicAuthInterceptor
    participant Sec as Spring Security Filter
    participant Ctrl as EmployeeController
    participant Svc as EmployeeServiceImpl
    participant Repo as EmployeeRepository / CompensationRepository
    participant DB as MySQL Database

    HR->>UI: Types "David" in search box & selects Status="Active"
    Note over UI: RxJS debounces input for 300ms
    UI->>Interceptor: Issue GET /api/v1/employees?search=David&status=ACTIVE
    Interceptor->>Sec: Attach Authorization: Basic hr_admin:hr_secret_123
    Sec->>Ctrl: Validate credentials & forward to controller
    Ctrl->>Svc: getEmployees(deptId=null, country=null, status="ACTIVE", search="David", pageable)
    Svc->>Repo: findAllFiltered(...)
    Repo->>DB: Execute SQL query with JOIN and WHERE UPPER(e.status) = 'ACTIVE'
    DB-->>Repo: Return matching Employee records
    Repo-->>Svc: Page<Employee>
    Svc-->>Ctrl: Map to Page<EmployeeResponseDTO>
    Ctrl-->>UI: Return 200 OK + JSON
    UI-->>HR: Display filtered search results in table

    HR->>UI: Clicks "View / Edit", modifies Base Pay to $120,000, clicks "Save"
    UI->>Interceptor: Issue PUT /api/v1/employees/16/salary with SalaryUpdateDTO
    Interceptor->>Sec: Attach Basic Auth Header
    Sec->>Ctrl: Validate & forward request
    Ctrl->>Svc: updateEmployeeSalary(id=16, updateDTO)
    Svc->>Repo: findWithCompensationById(16)
    Repo->>DB: Fetch Employee & linked Compensation
    Svc->>Svc: Recalculate Total CTC = BasePay + PF + OtherDeductions
    Svc->>Repo: save(employee)
    DB-->>Repo: Persist updated compensation & status
    Svc-->>Ctrl: Map to EmployeeResponseDTO
    Ctrl-->>UI: Return 200 OK + Updated DTO
    UI-->>HR: Refresh table & show success feedback
```

---

### User Journey B: Executive Views Global Headcount Map & Analytics

```mermaid
sequenceDiagram
    autonumber
    actor Exec as Executive / Admin
    participant UI as Angular Home / Analytics View
    participant Sec as Backend Security Filter
    participant Ctrl as AnalyticsController
    participant Svc as AnalyticsServiceImpl
    participant Repo as CompensationRepository
    participant DB as MySQL Database

    Exec->>UI: Opens Dashboard Home (http://localhost:4200)
    UI->>Sec: Parallel GET /api/v1/analytics/summary AND /api/v1/analytics/breakdown/country
    Sec->>Ctrl: Authenticate & route requests
    Ctrl->>Svc: getSummary() & getCountryBreakdown()
    Svc->>Repo: Execute JPQL active headcount & country breakdown queries
    Note over Repo: Filters active employees: WHERE UPPER(e.status) = 'ACTIVE'
    Repo->>DB: Run aggregation SQL queries (COUNT, SUM, AVG)
    DB-->>Repo: Return raw summary & country totals
    Repo-->>Svc: Return DTOs (SalaryAnalyticsDTO & List<CountrySummaryDTO>)
    Svc-->>Ctrl: Return responses
    Ctrl-->>UI: 200 OK + JSON Data
    Note over UI: Leaflet clears layer group & renders custom badge callout markers on world map
    UI-->>Exec: Display live interactive map with real active headcount per country
```

---

## 4. Entity Relationship & Data Model Specification

The database model is normalized to Third Normal Form (3NF) to guarantee data integrity across departments, positions, countries, and compensation records.

```mermaid
erDiagram
    DEPARTMENT ||--o{ EMPLOYEE : "belongs to"
    JOB_POSITION ||--o{ EMPLOYEE : "holds"
    COUNTRY ||--o{ EMPLOYEE : "stationed in"
    EMPLOYEE ||--o| COMPENSATION : "receives"
    EMPLOYEE ||--o{ EMPLOYEE_MONTHLY_LEAVE : "accumulates"

    EMPLOYEE {
        bigint id PK "Auto Increment"
        varchar emp_code UK "Unique Employee Code (EMP-XXXXX)"
        varchar first_name "First Name"
        varchar last_name "Last Name"
        varchar email UK "Unique Corporate Email"
        varchar phone_number "Contact Phone Number"
        bigint department_id FK "References departments(id)"
        bigint job_position_id FK "References job_positions(id)"
        char country_code FK "CHAR(3) References countries(country_code)"
        varchar status "ACTIVE / TERMINATED / INACTIVE"
        date date_of_joining "Joining Date"
    }

    COMPENSATION {
        bigint id PK "Auto Increment"
        bigint employee_id FK,UK "One-to-One FK to employees(id)"
        decimal base_pay "Base Annual Salary ($)"
        decimal pf_deduction "Provident Fund Deduction ($)"
        decimal other_deductions "Other Deductions ($)"
        int paid_leaves_allowance "Annual Paid Leaves (Days)"
        int sick_leaves_allowance "Annual Sick Leaves (Days)"
    }

    DEPARTMENT {
        bigint id PK "Auto Increment"
        varchar name "Department Name (Engineering, HR, etc.)"
        varchar code "Short Code (ENG, HR, FIN, MKT, SLS)"
    }

    JOB_POSITION {
        bigint id PK "Auto Increment"
        varchar title "Job Title"
        varchar code "Position Code"
    }

    COUNTRY {
        char country_code PK "CHAR(3) ISO Code (USA, CAN, GBR, etc.)"
        varchar country_name "Full Country Name"
        varchar currency_code "3-letter Currency Code (USD, EUR, GBP, etc.)"
    }

    EMPLOYEE_MONTHLY_LEAVE {
        bigint id PK "Auto Increment"
        bigint employee_id FK "References employees(id)"
        int year "Calendar Year"
        int month "Month Number (1-12)"
        int paid_leaves_taken "Paid Days Taken"
        int sick_leaves_taken "Sick Days Taken"
    }
```

---

## 5. Security & Authentication Architecture

Security is enforced at both frontend and backend boundaries:

1. **Frontend Credential Handling**:
   - `BasicAuthInterceptor` automatically injects HTTP Basic Credentials into every outgoing HTTP request header:
     $$\text{Authorization: Basic } \text{Base64}(\text{"hr\_admin:hr\_secret\_123"})$$
2. **Backend Spring Security Enforcement**:
   - Configured in `SecurityConfig.java`.
   - Protects all `/api/v1/**` endpoints.
   - Unauthenticated requests receive `401 Unauthorized`.
   - Cross-Origin Resource Sharing (CORS) is configured to permit `http://localhost:4200`.

---

## 6. High-Volume Batch Data Seeding (10,000 Records)

To support performance testing across enterprise-scale data sets, the backend includes an optimized bulk seeder (`SeederServiceImpl.java`):

- **Algorithm**: Generates 10,000 employees distributed randomly across 5 departments, 10 job positions, and 10 operating countries.
- **Realistic Salary Distribution**: Generates salary values adhering to Gaussian normal distribution benchmarks centered around $100,000 base pay.
- **Batch Processing Optimization**: Uses Hibernate batching (`spring.jpa.properties.hibernate.jdbc.batch_size=500`). The service processes records in chunks of 500, explicitly invoking `entityManager.flush()` and `entityManager.clear()` after each batch to prevent Java heap memory exhaustion.

---

## 7. Operational & Setup Guide

### 1. Prerequisites
- **Java Development Kit (JDK)**: OpenJDK 17 or 19
- **Node.js**: Node v18+ and npm
- **Database**: MySQL Server 8.0 running on `localhost:3306` with database name `acme_salary_db`

### 2. Running the Backend Server
```powershell
cd "d:\Salary Management System\salary-management-backend"
$env:JAVA_HOME = "C:\Users\ankit\.jdks\openjdk-19.0.2"
.\mvnw.cmd spring-boot:run
```
The backend API will start at `http://localhost:8080`.

### 3. Running the Frontend UI
```powershell
cd "d:\Salary Management System\salary-management-frontend"
npm start
```
The Angular UI will start at `http://localhost:4200`.

---

## 8. Summary of Component Files

| Project | File Path | Description |
| :--- | :--- | :--- |
| **Backend** | `src/main/java/.../entity/Employee.java` | Main Employee JPA Entity with annotations and relationships. |
| **Backend** | `src/main/java/.../entity/Compensation.java` | Compensation breakdown JPA Entity linked to Employee. |
| **Backend** | `src/main/java/.../dto/CountrySummaryDTO.java` | Regional headcount and salary summary record with constructor handling. |
| **Backend** | `src/main/java/.../repository/CompensationRepository.java` | Analytics JPQL queries with active employee status filtering. |
| **Backend** | `src/main/java/.../service/impl/SeederServiceImpl.java` | High-performance 10k batch database seeder implementation. |
| **Frontend** | `src/app/features/home/home.component.ts` | Executive homepage featuring Leaflet GIS map & stat overlay cards. |
| **Frontend** | `src/app/features/employee-directory/employee-directory.component.ts` | Employee directory data table with debounced search and edit modal. |
| **Frontend** | `src/app/features/analytics/analytics-dashboard.component.ts` | Financial charts dashboard powered by ng2-charts / Chart.js. |
| **Frontend** | `src/app/core/interceptors/basic-auth.interceptor.ts` | HTTP Basic Auth credential injection interceptor. |
