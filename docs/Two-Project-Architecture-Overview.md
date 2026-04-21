# Two-Project Architecture Overview

## Overview
The solution is built using two .NET 8 projects that work together to integrate **IRIS CARBON** with **Microsoft Teams** for document approval workflows.

- **IntegrationService** acts as the backend API layer.
- **TeamsApprovalBot** acts as the Microsoft Teams bot layer.

This separation helps keep the solution modular, scalable, and easier to maintain.

---

## Projects and Responsibilities

### 1. IntegrationService
**Role:** Backend API service that connects IRIS CARBON with Microsoft Teams and manages workflow operations.

**Responsibilities:**
- Create Teams groups and channels
- Manage memberships through Microsoft Graph API
- Trigger approval card sending
- Process approval and rejection actions
- Maintain workflow audit trail

**Key APIs:**
- `POST /api/teams/provision/company-group` – Create a Teams team for a company
- `POST /api/teams/provision/project-group` – Create a Teams team for a project
- `POST /api/teams/provision/project-channel` – Create a channel inside a Teams team
- `POST /api/teams/send-approval-card` – Send approval card request to TeamsApprovalBot
- `POST /api/workflow/approval-card` – Post approval card and supersede prior active cards
- `POST /api/workflow/approve` – Approve a workflow item
- `POST /api/workflow/reject` – Reject a workflow item
- `GET /api/workflow/audit` – Retrieve audit trail details

**Technology Stack:**
- ASP.NET Core 8
- MongoDB
- Microsoft Graph SDK v5
- ErrorOr pattern
- JWT Authentication

---

### 2. TeamsApprovalBot
**Role:** Microsoft Teams bot service that sends Adaptive Cards and handles user interactions inside Teams.

**Responsibilities:**
- Send approval Adaptive Cards to Teams channels
- Receive user actions such as Approve and Reject
- Validate incoming Teams actions
- Pass decisions back to IntegrationService

**Key APIs:**
- `POST /api/messages` – Bot Framework endpoint for Teams activities
- `POST /api/approval-card/send` – Send approval Adaptive Card to a Teams channel

**Adaptive Cards Used:**
- `approval-card.json` – Approval Required card with Approve, Reject, and Open in CARBON actions
- `completed-card.json` – Replacement card shown after final decision

**Technology Stack:**
- ASP.NET Core 8
- Microsoft Bot Framework 4.22
- AdaptiveCards Templating
- Newtonsoft.Json

---

## High-Level Architecture

```text
IRIS CARBON
    |
    v
IntegrationService (.NET 8 API)
    |-- Teams provisioning APIs
    |-- Workflow approval APIs
    |-- Audit trail management
    |
    v
TeamsApprovalBot (.NET 8 Bot Framework)
    |-- Sends Adaptive Cards to Teams
    |-- Receives Approve/Reject actions
    |
    v
Microsoft Teams
```

---

## End-to-End Workflow

```text
1. IRIS CARBON sends a request to IntegrationService
2. IntegrationService processes the request and calls TeamsApprovalBot
3. TeamsApprovalBot sends an Adaptive Card into a Teams channel
4. A reviewer clicks Approve or Reject in Microsoft Teams
5. Teams sends the action to TeamsApprovalBot
6. TeamsApprovalBot validates the action and prepares the decision
7. TeamsApprovalBot calls IntegrationService to record the response
8. IntegrationService updates workflow status and audit trail
```

---

## Current Implementation Status

| Feature | Status |
|---|---|
| Teams provisioning (create teams/channels/members) | Done |
| Approval card templates | Done |
| Proactive card sending | Done |
| Approve/Reject routing and validation | Done |
| Audit trail with integrity hashing | Done |
| Bot to IntegrationService callback | TODO |
| Card replacement after decision | TODO |
| Persistent conversation reference storage | TODO |

---

## Short Description
The architecture uses two separate services to support Microsoft Teams-based approval workflows. **IntegrationService** handles business logic, provisioning, workflow processing, and audit management. **TeamsApprovalBot** handles Microsoft Teams communication, Adaptive Card delivery, and user interaction processing. Together, they provide a structured and scalable integration approach for IRIS CARBON approval workflows.