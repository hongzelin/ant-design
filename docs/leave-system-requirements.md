# Leave Request System Requirements

## 1. Overview
This document defines the product requirements for a leave request system derived from the uploaded UI mockups. The system must support end-to-end leave management, including request creation, multi-level approvals, tracking, and CRUD operations on all relevant entities.

## 2. Scope
- Web-based internal system for employees and approvers.
- Covers annual leave but must be extensible to other leave types.
- Includes employee leave balance visibility, project attribution, attachment handling, and audit logging.

## 3. Actors & Permissions
- **Employee (Applicant)**: Create, view, edit, or withdraw own requests until final approval. View approval status and comments.
- **Approver (Level 1-5)**: Review requests assigned based on project/organization hierarchy. Approve, reject, transfer, or request edits.
- **Administrator/HR**: Manage leave type configuration, entitlements, approval chains, and perform overrides.

## 4. Business Rules
1. Leave balances follow yearly accrual with carry-over rules shown to the applicant at creation time.
2. Requests can be made by hour, half-day (morning/afternoon), or full-day segments.
3. Current year leave must be exhausted before borrowing from next year unless HR override is granted.
4. Each request must belong to a project or cost center when required by policy.
5. Up to five sequential approver levels are supported; at least one approver is mandatory.
6. Attachments are optional but required when the leave type requests proof.
7. Travel checkbox indicates if the leave involves business travel; selecting it can trigger additional workflow (e.g., travel desk notification).

## 5. Data Model & Fields

### 5.1 Employee
| Field | Type | Description |
| --- | --- | --- |
| employee_id | string | Unique ID (e.g., 0027009318). |
| name | string | Employee full name. |
| location | string | Work location (e.g., Guangzhou). |
| department | string | Owning department. |
| manager_id | string | Direct manager. |

### 5.2 Leave Balance
| Field | Type | Description |
| --- | --- | --- |
| employee_id | string | FK to Employee. |
| year | int | Leave year (e.g., 2025). |
| carry_over_days | decimal(5,2) | Days carried from previous year. |
| accrued_days | decimal(5,2) | Current year entitlement. |
| used_days | decimal(5,2) | Approved leave consumed. |
| available_days | decimal(5,2) | Remaining leave. |

### 5.3 Leave Request
| Field | Type | Description / Rules |
| --- | --- | --- |
| request_id | string | Unique running number (e.g., 0027009318-2025-0001). |
| employee_id | string | FK to Employee. |
| leave_type | enum | Annual Leave, Sick Leave, etc. Drives validation rules. |
| request_mode | enum | `BY_HOUR`, `BY_INTERVAL`. |
| start_date | date | Leave start date. |
| end_date | date | Leave end date (same day allowed). |
| am_hours | decimal(3,1) | Hours in morning segment when using hourly mode. |
| pm_hours | decimal(3,1) | Hours in afternoon segment. |
| total_hours | decimal(4,1) | Calculated total. |
| total_days | decimal(4,2) | Derived (hours/8). |
| duration_slot | enum | `AM`, `PM`, `FULL` for day-based requests. |
| project_id | string | Project/initiative selected by applicant. |
| remarks | text | Applicant remarks. |
| travel_flag | boolean | Indicates business travel. |
| attachment_ids | array | FK to attachment table. |
| status | enum | `DRAFT`, `SUBMITTED`, `UNDER_REVIEW`, `APPROVED`, `REJECTED`, `CANCELLED`. |
| submit_time | datetime | Timestamp of submission. |
| approval_completed_time | datetime | When final decision is made. |
| created_at / updated_at | datetime | Audit timestamps. |

### 5.4 Approval Chain
| Field | Type | Description |
| --- | --- | --- |
| approval_id | string | Unique ID. |
| request_id | string | FK to Leave Request. |
| level | int | Sequence (1-5). |
| approver_id | string | FK to Employee. |
| approver_role | string | Role (e.g., Project Director, HR). |
| decision | enum | `PENDING`, `APPROVED`, `REJECTED`, `TRANSFERRED`, `REQUEST_INFO`. |
| decision_time | datetime | Timestamp of action. |
| comments | text | Approver remarks. |
| attachment_ids | array | Supporting documents added by approver. |

### 5.5 Attachment
| Field | Type | Description |
| --- | --- | --- |
| attachment_id | string | Unique file reference. |
| request_id | string | FK to Leave Request. |
| file_name | string | Original name. |
| file_type | string | MIME type. |
| file_size | int | Size in bytes (limit e.g., 10 MB). |
| uploader_id | string | Employee/Approver who uploaded. |
| uploaded_at | datetime | Timestamp. |

### 5.6 Audit Log
| Field | Type | Description |
| --- | --- | --- |
| log_id | string | Unique entry. |
| entity_type | enum | Request, Approval, Attachment. |
| entity_id | string | FK to entity. |
| action | enum | CREATE, UPDATE, DELETE, APPROVE, REJECT, CANCEL. |
| actor_id | string | User performing action. |
| action_time | datetime | Timestamp. |
| payload | json | Changed fields snapshot. |

## 6. Functional Requirements (CRUD)

### 6.1 Leave Request Listing (Read)
- **Filters**: status (all/pending/approved), start date (apply), end date, leave type, project, applicant, approver.
- **Columns**: applicant (ID + name), start date, end date, AM hours, PM hours, total length (days), leave type, project, approver levels 1-5, remarks, submission date, approval date, decision result, attachments indicator, status.
- **Actions**: open detail view, export to CSV, batch selection for admin actions.

### 6.2 Create Leave Request (Create)
- Prefill employee info and leave balance summary panel.
- Required fields: leave type, request mode, start/end date, duration slot or hours, approver selection (at least one level).
- Validation: ensure hours align with chosen mode, dates within allowable range, total duration <= available balance unless override flag set.
- Submitting transitions status to `SUBMITTED` and instantiates approval chain records per level.

### 6.3 Update Leave Request (Update)
- Applicant can edit requests in `DRAFT` or `REJECTED` state. Fields editable: date range, duration, leave type, project, remarks, attachments, approver chain.
- Editing resets approval chain and notifications.

### 6.4 Cancel Leave Request (Delete/Soft Delete)
- Applicant may cancel requests in `SUBMITTED` or `APPROVED` states before leave start date. Status becomes `CANCELLED`, leave balance restored.
- Approvers and HR can force close requests with audit note.

### 6.5 Approval Processing (Read/Update)
- Approvers receive queue of pending requests filtered by level and project.
- Actions per request: `Approve`, `Reject`, `Transfer` (assign different approver), `Request Info`.
- Adding comments is mandatory on reject/transfer.
- Approval completion rules: once current level approves, next level becomes active; final approval sets request status to `APPROVED`.
- Rejection sets status to `REJECTED` and halts workflow.
- Transfer updates approver_id for the same level and logs action.

### 6.6 Attachment Management (CRUD)
- Upload within request form or approval modal (drag-and-drop and browse).
- Support download and preview according to file type.
- Enforce virus scan hook and size limits.

### 6.7 Notifications
- Email/IM notifications triggered on submission, approval decisions, rejection, cancellation, and requested information.
- Include request summary and action links.

### 6.8 Reporting & Audit
- Export filters result set to CSV/XLS.
- Admin dashboard showing leave usage per year and outstanding approvals.
- Audit log accessible by HR with filtering by date, actor, action.

## 7. API/CRUD Matrix
| Entity | Create | Read | Update | Delete |
| --- | --- | --- | --- | --- |
| Leave Request | POST `/api/leave-requests` | GET `/api/leave-requests` (list/detail) | PATCH `/api/leave-requests/{id}` | POST `/api/leave-requests/{id}/cancel` (soft delete) |
| Approval | Auto-create via workflow | GET `/api/leave-requests/{id}/approvals` | PATCH `/api/approvals/{id}` for decisions | N/A (immutable history) |
| Attachment | POST `/api/attachments` | GET `/api/attachments/{id}` | PATCH metadata (optional) | DELETE `/api/attachments/{id}` |
| Leave Balance | Managed via HR system | GET `/api/leave-balances/{employee_id}` | PATCH for manual adjustments | N/A |
| Audit Log | Auto-create | GET `/api/audit-logs` | N/A | N/A |

## 8. Non-Functional Requirements
- **Security**: Role-based access control, data encryption at rest and in transit, attachment virus scanning.
- **Performance**: List view must return first page results within 2 seconds for up to 5000 records overall.
- **Scalability**: Support 10,000 active users with auto-scaling API tier.
- **Reliability**: 99.5% uptime, transactional consistency between leave balance updates and request status changes.
- **Localization**: UI labels and leave type names must support multiple languages (e.g., Chinese, English).
- **Auditability**: Every change logged with actor, timestamp, and payload.

## 9. Open Questions
1. Do different leave types require distinct approval chains?
2. Should travel flag trigger integration with travel booking workflow?
3. Are partial-day requests allowed on weekends/holidays if employee is scheduled to work?

