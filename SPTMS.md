# SmartPlan


## 1. Document Control

| Field                      | Value                                     |
| -------------------------- | ----------------------------------------- |
| **Version**                | 1.2                                       |
| **Author / Owner**         | TBD                                       |
| **Date Created / Updated** | 2025-11-07                                |
| **Status**                 | Draft                                     |
| **Change Log**             | (Version / Summary / Author / Date) — TBD |


## 2. Project Overview

**Project Name:** SmartPlan

**Description**  
SmartPlan is a centralized task and goal management platform that enables hierarchical organizations to plan, assign, execute, and monitor work efficiently. It connects tasks, objectives and reporting so teams at all levels have clearly visible responsibilities, aligned goals and reliable performance metrics.

**Business Context**  
Organizations that use fragmented task management methods suffer reduced visibility, lower accountability, and inefficient collaboration. SmartPlan provides a unified platform that reduces fragmentation, improves traceability and supports role-based governance across organizational units.

**Objectives & Key Results (OKRs)**

1. Improve organizational efficiency through centralized task management.
    
2. Increase visibility and accountability at all operational levels.
    
3. Improve alignment between day-to-day tasks and organizational objectives.
    


## 3. Scope

**In-Scope**

- Configurable organizational hierarchy and role-based access control.
    
- Full task lifecycle: creation, assignment, collaboration, tracking, and closure.
    
- Task visualization: List, Kanban, Calendar, and Gantt views (client-side rendering powered by backend data).
    
- Role-based dashboards and progress reports.
    
- Notification center (in-app and email).
    
- Goal / OKR tracking with traceability to tasks.
    
- Core integrations: directory synchronization, email ingestion, calendar sync (phased), chat notifications, and document attachments.
    
- RESTful API surface for clients and external integrations.
    

**Major Deliverables**

- Web application (SmartPlan Web).
    
- Backend API service.
    
- Admin configuration and reporting dashboards.
    

## 4. Functional Requirements

**User Management**

- Custom user profiles and organizational mapping.
    
- Role-based permissions mapped to organizational units.
    
- Admin UI for role assignment and org structure configuration.
    

**Task Lifecycle**

- Task creation, editing, assignment, comments, status updates and closure.
    
- Subtasks, optional dependencies, and task templates.
    
- Recurring task generation via scheduled job.
    
- Attachments metadata stored and served via object storage integration.
    

**Collaboration & Assignment**

- Multi-assignee/co-responsible tasks and follower/observer roles.
    
- Comments with mentions and threaded activity feed.
    
- Workload snapshot used to avoid assignment conflicts.
    

**Views & Visualization**

- Backend supports payloads for List, Kanban, Calendar, and Gantt views; clients render UI.
    
- Saved filters and quick search across indexed fields.
    

**Notifications**

- In-app and email notifications for assignments, mentions and state changes.
    
- Configurable reminders and periodic digests.
    

**Dashboards**

- Role-scoped dashboards with KPIs, overdues, performance and OKR progress.
    
- Exportable reports for selected views.
    

**Goal / OKR Management**

- Objectives + Key Results entities with linkable tasks for traceability.
    
- Progress computation and goal alignment views.
    

**Integrations**

- Email-to-task ingestion, calendar sync, chat notifications, document attachments via cloud drives, and webhook hooks for external systems.


## 5. Non-Functional Requirements

Performance, scalability, availability, reliability, security, usability, maintainability and compliance requirements remain as previously defined and are not changed in this update.


## 6. System Concepts — Hierarchy & Role Model

### 6.1 Generic Organization Model

- **Organization**  the top-level company entity.
    
- **OrgUnit**  a flexible organizational unit model representing divisions, departments, teams or any custom grouping. OrgUnits form a tree with arbitrary depth.
    
- **Users**  people who belong to an Organization and are members of one or more OrgUnits (membership can be many-to-many).
    
- **RoleAssignment**  a mapping of a user to a role scoped to an OrgUnit (or to the whole Organization). A RoleAssignment contains: `(user_id, role_name, orgunit_id, assigned_by, assigned_at, is_active)`.
    

### 6.2 Example Hierarchy (illustrative)

```
Organization: Acme Corp
└── OrgUnit: Division A
    ├── OrgUnit: Department A1
    │   └── OrgUnit: Team A1-1
    │       ├── Team Lead
    │       └── Staff Members
    └── OrgUnit: Department A2
        └── OrgUnit: Team A2-1
```

### 6.3 User → Group Mapping (Django Groups)

Each user is mapped to Django permission groups to reflect role assignments. The recommended seeded groups are:

- `SuperAdmin` — platform-level administrators.
    
- `OrgAdmin` — organization-level administrators.
    
- `Manager` — department/division managers.
    
- `TeamLead` — team leads.
    
- `Staff` — regular contributors.
    

Mapping behavior:

- A RoleAssignment creates or updates group membership. If a user has `RoleAssignment(role='Manager', orgunit=Department A1)`, the system places the user in the `Manager` Django group and records the scope (`OrgUnit=Department A1`) in the RoleAssignment table. Authorization checks consult RoleAssignment scope rather than group membership alone; groups provide permission defaults and convenient UI/administration.
    


## 7. System Architecture

### 7.1 Layered Architecture Overview

```
[ Frontend Layer ]
  - Web & Mobile UI (task boards, dashboards, OKR boards)

[ Backend Layer ]
  - Business logic: identity, tasks, policies, notifications, analytics

[ Data Layer ]
  - Relational DB (transactional data)
  - Object storage (attachments)
  - Immutable event ledger (audit + analytics)

[ AI Layer (optional) ]
  - Summaries, retrieval and scope-enforced assistant

[ Integration Layer ]
  - Directory sync, email ingestion, calendar connectors, chat notification outlets
```

### 7.2 Core System Services & Interaction Flow

**Core System Services**

- **Identity & Access Service** — manages users, OrgUnits, RoleAssignments and enforces scoping rules.
    
- **Task & Workflow Service** — implements task model, lifecycle transitions, templates and recurrence rules.
    
- **Notification & Event Service** — records TaskEvent entries, drives in-app notifications and email digests.
    
- **Analytics & Snapshot Service** — computes aggregated metrics and OrgSummary/UserSummary snapshots for fast read paths.
    
- **AI Service** (optional) — generates organization-scoped insights using snapshots and event documents.
    

**Interaction Flow**

1. User performs action in frontend (create task, comment, etc.).
    
2. Backend validates action against Policy & Permission Service and persists changes to transactional tables.
    
3. Changes are appended to the immutable TaskEvent ledger.
    
4. Notification & Event Service emits notifications and records audit entries.
    
5. Aggregation jobs consume events and update DailyMetrics and OrgSummary snapshots.
    
6. Dashboards and API clients read snapshots for low-latency responses.
    
7. Optional AI Service consumes snapshots and (when enabled) provides scoped recommendations.
    

## 8. Backend Architecture (Project Structure & App Features)

### 8.1 Project Root & App Tree

```
sptms_api/                # Django project container
├─ sptms_api/             # Django settings, urls, asgi/wsgi
├─ core/                  # Cross-cutting business logic and utilities
├─ accounts/              # User, Organization, OrgUnit, RoleAssignment
├─ tasks/                 # Task model, templates, comments, attachments, events
├─ analytics/             # DailyMetrics, OrgSummary, aggregation jobs
└─ ai/                    # Optional: ingestion, retrieval, chat orchestration (disabled by default)
```

### 8.2 App Responsibilities and Key Features

**core**

- Central policy checks (`can_assign`, `is_ancestor`), notification primitives, event registry, and summary snapshot orchestration.
    
- Shared utilities used by other apps (serializers, base models, common enums).
    

**accounts**

- Models: `User`, `Profile`, `Organization`, `OrgUnit`.
    
- RoleAssignment service for role lifecycle, seeding default groups, and admin interfaces.
    
- Membership mapping to Django groups and admin management views.
    

**tasks**

- Models: `Task`, `TaskTemplate`, `TaskComment`, `Attachment` (metadata), `TaskTimeLog`, `TaskEvent`.
    
- Business logic: assignment rules, status transitions, subtask handling, and template expansion.
    
- APIs for task CRUD, comments, attachments and assignment operations.
    

**analytics**

- Models: `DailyMetrics`, `OrgSummary`, `UserSummary`.
    
- Aggregation management commands that consume TaskEvent ledger and update snapshots for dashboards and AI ingestion.
    

**ai** (optional)

- Ingestion pipelines (OrgSummary + event documents), lightweight retrieval mechanisms, and a scoped assistant interface. Disabled by default and enabled per organization with governance and audit trails.
    

### 8.3 Role / Permission Concepts and Enforcement

- Permissions derive from `RoleAssignment` scope: a role assigned to an OrgUnit covers that OrgUnit and all descendants unless explicitly restricted.
    
- Permission check examples:
    
    - `can_assign(actor, assignee, target_orgunit)` — determines whether the actor may assign a task to the assignee within the target scope.
        
    - `can_edit_task(actor, task)` — checks ownership, role scope and explicit delegation flags.
        
- Group seeding: default Django groups (`SuperAdmin`, `OrgAdmin`, `Manager`, `TeamLead`, `Staff`) are created via a migration so all environments share the same starting security model.
    

### 8.4 Authentication, Sessions & Auditing

- Authentication tokens and session mechanisms are provided by the platform; session state is stateless on the server side where possible.
    
- All administrative actions, reassignments, role changes and AI-initiated actions (if enabled) are stored in the TaskEvent ledger and a separate AuditLog table for compliance.
    

### 8.5 Background Scheduling Strategy

- Background tasks such as snapshot aggregation, recurring task creation and email digests are implemented as management commands scheduled by the host environment (cron or cloud scheduler).
    
- No message broker or worker queue is required in the initial architecture. The scheduling approach minimizes operational complexity and supports predictable execution patterns. If required later, a worker queue can be introduced as an extension.
    

## 9. Data Architecture

### 9.1 Core Data Types & Tables

- **Transactional:** Users, Organizations, OrgUnits, RoleAssignments, Tasks, Comments, Attachments, TaskTimeLogs.
    
- **Event ledger:** TaskEvent (append-only): event_type, payload, actor_id, task_id, created_at.
    
- **Aggregates / snapshots:** DailyMetrics, OrgSummary, UserSummary for fast read scenarios and AI prompts.
    
- **Retention / backup:** Soft-delete semantics (archival after configurable period), DB backups per provider policy.
    

### 9.2 Metrics by Role

- **System Administrator:** platform-level totals (organizations, active users, overall task volume).
    
- **Organization Administrator:** org-level KPIs (task volume by OrgUnit, completion rate, SLA violations).
    
- **Manager:** department KPIs (overdue rate, average completion time, team workload).
    
- **Team Lead:** team-level operational metrics (active tasks, blocked tasks, member utilization).
    
- **Staff:** personal metrics (assigned tasks, due soon, time logged).
    

### 9.3 Data Flow

- User actions → transactional write → TaskEvent append → aggregator jobs consume events → DailyMetrics / OrgSummary updated → dashboards read snapshots → optional AI consumes snapshots for insights.
    

## 10. APIs & Integrations

- The backend exposes versioned REST APIs for all domain interactions (users, org units, tasks, attachments, events and snapshots).
    
- Integration channels are standardized connectors: directory synchronization, inbound email → task ingestion, calendar sync, chat notifications, and webhooks for external systems.
    
- Integrations write canonical events into the TaskEvent ledger so external and internal sources use the same canonical event model.
    

## 11. Security & Privacy

- Role-scoped access control enforces least privilege.
    
- Transport encryption for all network traffic.
    
- Audit logs for role changes, task reassignments and data exports.
    
- Configurable data retention and deletion flows to meet regulatory requirements.
    

## 12. Infrastructure & Deployment

- Environments: Development → Staging → Production with CI/CD for builds, tests and deployments.
    
- Object storage for attachments and managed relational DB for transactional data.
    
- Scheduled management commands orchestrate background jobs; scheduler type is chosen according to hosting environment.
    


## 13. Testing & Quality Assurance

- Unit tests for models and policy logic.
    
- Integration tests for API flows and aggregations.
    
- End-to-end tests for critical user journeys such as task creation, assignment and reporting.
    
- Performance benchmarks for snapshot queries and dashboard payloads.
    

## 14. Monitoring & Observability

- Instrumentation for API latency, error rates, job success/failure and resource utilization.
    
- Alerts for failed scheduled jobs, abnormal error spikes and infrastructure issues.
    
- Retention and archival of logs for operational and compliance purposes.
    

## 15. Execution Plan — 9 weeks + 15-day AI 


| Week                     | Phase                         | Focus / Deliverables                                                                                             |
| ------------------------ | ----------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **1**                    | Project Initialization        | Architecture confirmation, schema design, role setup, initial migrations.                                        |
| **2**                    | Core Hierarchy & Roles        | Implement Organization, OrgUnit, User model and RoleAssignment seeding.                                          |
| **3**                    | Task Lifecycle Core           | Implement Task, templates, assignment rules and subtask support.                                                 |
| **4**                    | Collaboration & Notifications | Comments, mentions, TaskEvent ledger and notification primitives.                                                |
| **5**                    | Aggregation & Snapshots       | DailyMetrics and OrgSummary generator (management commands).                                                     |
| **6**                    | Dashboards & Metrics          | Dashboard endpoints, saved views, role-scoped access.                                                            |
| **7**                    | Integrations & Attachments    | Email ingestion, calendar hooks (phased), Hopalase attachment handling.                                          |
| **8**                    | Testing & QA                  | Integration tests, API docs, staging deployment and validation.                                                  |
| **9**                    | Stabilization & Hardening     | Address findings, optimize snapshot flows, prepare launch.                                                       |
| **+15 Days (AI Sprint)** | AI Insight Layer              | Extract snapshots & events, implement retrieval & summarization, scoped assistant integration and audit logging. |
|                          |                               |                                                                                                                  |


## 16. Appendices

References:

Source: Task Management – Requirement Description.pdf