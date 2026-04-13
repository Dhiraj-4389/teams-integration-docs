# Proposed Solution Document  
## Microsoft Teams Integration for IRIS CARBON Disclosure Approval Workflow

**Document Version:** 1.0  
**Prepared For:** Management Review  
**Prepared By:** Solution Analysis  
**Date:** 2026-04-13  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)  
2. [Problem Statement](#2-problem-statement)  
3. [Business Context](#3-business-context)  
4. [Business Need and Strategic Objectives](#4-business-need-and-strategic-objectives)  
5. [Detailed Functional Requirements](#5-detailed-functional-requirements)  
   - [5.1 Organization-Level Teams Channel and Approval Cards](#51-organization-level-teams-channel-and-approval-cards)  
   - [5.2 Stale-Card Protection](#52-stale-card-protection)  
   - [5.3 Audit-Grade Approval Logging](#53-audit-grade-approval-logging)  
   - [5.4 Change Diff Digest Card](#54-change-diff-digest-card)  
   - [5.5 Validation and Exception Alerting](#55-validation-and-exception-alerting)  
   - [5.6 Workflow State Requirements](#56-workflow-state-requirements)  
   - [5.7 REST API Requirements](#57-rest-api-requirements)  
   - [5.8 Acceptance Criteria Summary](#58-acceptance-criteria-summary)  
6. [Integration Options Considered](#6-integration-options-considered)  
7. [Cross-Tenant Deployment and Consent Considerations](#7-cross-tenant-deployment-and-consent-considerations)  
8. [Comparative Capability Assessment](#8-comparative-capability-assessment)  
9. [Formal Assessment of Integration Options](#9-formal-assessment-of-integration-options)  
   - [9.1 Teams App + Bot Framework](#91-teams-app--bot-framework)  
   - [9.2 Microsoft Graph App-only](#92-microsoft-graph-app-only)  
   - [9.3 Incoming Webhook](#93-incoming-webhook)  
   - [9.4 Microsoft Teams Workflows](#94-microsoft-teams-workflows)  
   - [9.5 Power Automate](#95-power-automate)  
   - [9.6 Email-to-Teams](#96-email-to-teams)  
   - [9.7 Comparative Summary](#97-comparative-summary)  
   - [9.8 Recommended Best-Suited Option](#98-recommended-best-suited-option)  
10. [Proposed Solution Architecture](#10-proposed-solution-architecture)  
11. [Proposed End-to-End Operating Model](#11-proposed-end-to-end-operating-model)  
12. [Business, Technical, and Compliance Benefits](#12-business-technical-and-compliance-benefits)  
13. [Constraints, Assumptions, and Risks](#13-constraints-assumptions-and-risks)  
14. [Final Recommendation](#14-final-recommendation)  
15. [Appendices](#15-appendices)  
   - [15.1 Appendix A – Cross-Tenant Feasibility Table](#151-appendix-a--cross-tenant-feasibility-table)  
   - [15.2 Appendix B – Capability Comparison Table](#152-appendix-b--capability-comparison-table)  

---

## 1. Executive Summary

IRIS CARBON’s Disclosure Management module currently supports approval processing through the CARBON web application. Although functionally complete, the present model requires approvers to leave Microsoft Teams, open the CARBON application, locate the relevant document section, review changes, and then take an approval decision. This introduces repeated context switching, slows down decision-making, and reduces efficiency across high-volume filing cycles.

Given that Microsoft Teams is already the primary communication and collaboration platform for many CARBON customers, there is a clear strategic opportunity to move the approval interaction layer into Teams while preserving CARBON as the authoritative system for workflow, authorization, validation, and audit.

This document evaluates the principal integration options available for Microsoft Teams and assesses their suitability against CARBON’s enterprise-grade requirements, including:

- organization-level channel integration  
- secure approval and rejection actions  
- Adaptive Cards  
- version-aware stale-card validation  
- audit-grade decision logging  
- change-diff visibility  
- validation and exception alerting  
- cross-tenant enterprise deployment  

Following detailed analysis, the recommended solution is:

**Microsoft Teams App with Bot Framework integration as the primary interaction model, supported by Microsoft Graph for provisioning and channel-related operations where required.**

This approach is recommended because it is the only option that fully aligns with CARBON’s functional, governance, security, and auditability requirements. It provides the control and extensibility necessary for an enterprise disclosure workflow while enabling a modern Teams-based user experience.

For customer-owned Microsoft Teams environments, this solution should be positioned as a **customer-admin-onboarded enterprise integration model**, as a no-admin-consent cross-tenant rollout cannot be assumed as a reliable standard across enterprise customers.

---

## 2. Problem Statement

IRIS CARBON’s current disclosure approval process is web-centric. Users involved in the filing lifecycle—particularly approvers—must leave their collaboration environment in Microsoft Teams and move into the CARBON application in order to take action on pending sections.

This operating model introduces several material inefficiencies:

- repeated switching between collaboration and transaction systems  
- time lost in navigation and context recovery  
- reduced approval velocity across multiple sections and documents  
- limited visibility of what changed unless users open the detailed web view  
- reactive, rather than real-time, handling of validation and exception scenarios  
- increased risk of stale approvals if content changes after approval request generation  

In practical terms, approvers often manage 20–30 sections across multiple filing artifacts. Repeating the same navigation-heavy process for each section creates cumulative delay and operational friction.

The core business problem is therefore not a lack of workflow capability within CARBON, but a lack of workflow accessibility in the collaboration surface where users are already working. The solution must address that usability gap without compromising compliance, authorization control, version integrity, or audit traceability.

---

## 3. Business Context

IRIS CARBON Disclosure Management supports complex financial and regulatory filing workflows involving multiple roles, including:

- Preparer  
- Reviewer  
- Approver  
- Controller  

These users collaborate on disclosure content that is subject to workflow controls, content validation, audit requirements, and final lock controls. The platform therefore operates in a domain where governance and traceability are as important as usability.

At the same time, customer behavior has increasingly centered around Microsoft Teams as the day-to-day operational platform for collaboration, discussion, and issue triage. While communication occurs in Teams, the final workflow action still remains within CARBON. This separation between collaboration and transaction introduces friction and delays.

The proposed integration must therefore meet two principles simultaneously:

1. **Microsoft Teams must become the user interaction surface for approvals and alerts**  
2. **CARBON must remain the system of record for state, authorization, versioning, validation, and audit**  

This distinction is fundamental to ensuring a secure and compliant enterprise architecture.

---

## 4. Business Need and Strategic Objectives

### 4.1 Business Need

The business need is to streamline the approval lifecycle by enabling users to review and act on disclosure approvals directly within Microsoft Teams, thereby reducing approval latency and improving collaboration effectiveness, while preserving CARBON’s governance model.

### 4.2 Strategic Objectives

The proposed integration should achieve the following objectives:

1. Reduce approval turnaround time by minimizing context switching  
2. Surface approval requests directly within organization-specific Teams channels  
3. Enable secure inline approval and rejection from Teams  
4. Provide reviewers with sufficient change visibility to make informed decisions  
5. Surface validation and exception conditions in real time within the collaboration channel  
6. Prevent stale approvals through version-aware validation  
7. Capture all Teams-originated decisions in an immutable audit trail  
8. Support enterprise deployment in customer-owned Teams tenants  
9. Preserve CARBON as the authoritative source for workflow and compliance logic  

---

## 5. Detailed Functional Requirements

### 5.1 Organization-Level Teams Channel and Approval Cards

The solution must allow a CARBON administrator to associate an organization with a specific Microsoft Teams channel. This configuration should persist the following identifiers:

- organization identifier  
- Team ID  
- Channel ID  

The mapping should be stored in CARBON configuration and retrievable through an administrative interface or API.

When a section transitions into the `PENDING_APPROVAL` state, a Teams bot should proactively post an Adaptive Card into the configured organization channel. The approval card should display:

- section name  
- document name  
- document version  
- last editor  
- last edited timestamp  
- current workflow state  
- actions:
  - Approve  
  - Reject  
  - Open in CARBON  

All user actions taken from the card must be routed to CARBON for validation and execution.

Organizational isolation is mandatory. Cards for one organization must never appear in another organization’s Teams channel.

### 5.2 Stale-Card Protection

Approval actions in Teams must be protected against stale execution. Each approval card should therefore include hidden metadata sufficient to validate its freshness, including:

- section version hash  
- last modified timestamp  
- card issued timestamp  
- card instance identifier  

At the time of user action, CARBON must validate:

- the section is still in `PENDING_APPROVAL`
- the version hash on the card matches the current persisted section version
- the card remains within its permitted freshness threshold

Where the version does not match, the action must be rejected and the card should indicate that the section has changed since the card was issued. A refreshed card may then be posted.

Where a card has exceeded the inactivity threshold, such as 24 hours, action buttons should be disabled and a Refresh mechanism should be provided.

Where a new approval card is posted for the same section, the prior card must be updated as superseded and made non-actionable.

### 5.3 Audit-Grade Approval Logging

Every approval or rejection originating in Teams must be recorded in an immutable, append-only audit trail. The audit record must include the following mandatory fields:

- approverUserId  
- displayName  
- tenantId  
- serverTimestamp (UTC)  
- decision  
- sectionId  
- documentId  
- documentVersion  
- sectionVersionHash  
- previousState  
- newState  
- requestId / correlationId  
- sourceChannel = Teams  

Optional trace fields may include:

- Teams conversationId  
- Teams messageId  
- cardInstanceId  

Rejections must require a mandatory reason. Approval comments may be optional, depending on business rules.

For tamper evidence, each audit payload should be hashed using SHA-256 across a canonical sorted representation plus a server-side secret.

No update or delete should be permitted on audit records. Any correction must be inserted as a new audit entry referencing the original.

### 5.4 Change Diff Digest Card

When a section enters `PENDING_APPROVAL`, CARBON should compute the difference between the current section version and the last approved version.

The approval experience in Teams should provide:

- a compact summary of changes  
- counts of tables changed  
- tagged value changes  
- paragraph additions, modifications, or deletions  
- attribution of the latest editor  
- timestamp of the latest changes  
- expandable detail where appropriate  
- a deep link to the full detailed diff view in CARBON  

Where change volume is high, the card may display a top-level summary with overflow handled by linking to CARBON.

Where no content change exists since the previous approval, the card should clearly indicate that no substantive content changes were detected.

### 5.5 Validation and Exception Alerting

Validation and exception events must be surfaced proactively into the configured Teams channel. Triggers include:

- XBRL validation failure  
- totals mismatch  
- roll-forward mismatch  
- iXBRL issue  
- cross-table consistency mismatch  

Each validation alert card should display:

- alert type  
- severity  
- impacted section, table, or fact  
- issue description  

Potential actions may include:

- Assign Owner  
- Open Issue in CARBON  
- Re-run Validation  
- Mark as Accepted Exception  

Accepted exception handling must require justification and must be audit logged.

Duplicate unresolved validation issues should not generate duplicate cards. Instead, the existing card should be updated. Once a validation issue is corrected, the Teams card should reflect resolution status.

### 5.6 Workflow State Requirements

The solution must support the following states:

- DRAFT  
- IN_REVIEW  
- PENDING_APPROVAL  
- APPROVED  
- LOCKED  

Teams approval cards should only be issued when a section transitions into `PENDING_APPROVAL`.

Card actions are valid only while the section remains in `PENDING_APPROVAL`. Any state divergence at action time must cause the request to be treated as stale or invalid.

A requirement clarification remains necessary because one part of the specification indicates that rejection returns the section to `DRAFT`, while one acceptance criterion refers to a `REJECTED` state. This discrepancy should be resolved prior to implementation.

### 5.7 REST API Requirements

The integration requires the following APIs:

- `POST /api/teams/channels`  
- `DELETE /api/teams/channels/{orgId}`  
- `POST /api/teams/cards/approval`  
- `POST /api/teams/cards/action`  
- `POST /api/teams/cards/validation-alert`  
- `GET /api/audit/approvals`  
- `GET /api/audit/{id}/verify`  

### 5.8 Acceptance Criteria Summary

The solution must satisfy the documented acceptance expectations, including:

- successful organization-to-channel registration  
- proactive posting of approval cards within expected time windows  
- correct handling of approved and rejected decisions  
- mandatory reject reason enforcement  
- unauthorized action denial  
- stale-card rejection and refresh behavior  
- handling of already-actioned items  
- inactivity-based card expiry  
- complete and immutable audit capture  
- paginated and filterable audit retrieval  
- diff summary presentation  
- validation alert creation, update, and resolution  
- strict organization isolation  
- successful end-to-end workflow completion  

---

## 6. Integration Options Considered

The following Microsoft Teams integration models were assessed:

1. Teams App + Bot Framework  
2. Microsoft Graph App-only  
3. Incoming Webhook  
4. Microsoft Teams Workflows  
5. Power Automate  
6. Email-to-Teams  

Each option was assessed against CARBON’s specific enterprise requirements, with particular emphasis on:

- approval interaction quality  
- secure callback handling  
- identity and authorization support  
- stale-card and lifecycle control  
- auditability  
- document review capability  
- cross-tenant enterprise suitability  
- long-term extensibility  

---

## 7. Cross-Tenant Deployment and Consent Considerations

### 7.1 Enterprise Deployment Reality

A key design consideration is that CARBON customers operate within their own Microsoft 365 and Teams tenants. Any solution intended to function directly within a customer’s Teams environment must therefore respect the governance and controls enforced by that customer’s tenant administrators.

### 7.2 Consent and Governance Implications

In cross-tenant scenarios, the following factors are controlled by the customer organization:

- availability of custom Teams applications  
- permission to install or access bots  
- user consent policies  
- guest/external user restrictions  
- organizational governance controls  
- Teams application setup policies  

As a result, it is not operationally sound to assume that a custom Teams integration can be deployed into customer environments on a user-consent-only basis.

### 7.3 Strategic Implication

The proposed solution should be framed as a **customer-admin-onboarded enterprise integration**, with customer administrator participation expected during setup and tenant enablement.

This is a normal and acceptable operating model for enterprise SaaS integrations and should not be viewed as a design weakness. Rather, it reflects appropriate alignment with enterprise governance expectations.

---

## 8. Comparative Capability Assessment

### 8.1 Cross-Tenant Feasibility Table

| Option | Works in customer’s Teams tenant | Needs customer admin involvement | Supports secure approval workflow | Supports Document Review | Supports Adaptive Cards | Supports inline approve/reject callback | Enterprise suitability for requirement | Notes |
|---|---|---|---|---|---|---|---|---|
| Teams App + Bot Framework | Yes | Yes, typically required | Yes | Yes | Yes | Yes | Best | Most suitable for secure, governed, interactive workflow |
| Microsoft Graph App-only | Partial | Yes | Partial / No alone | Partial | Limited | No | Partial | Appropriate as a supporting technology, not as the primary interaction model |
| Incoming Webhook | Yes, if enabled | Usually yes | No | No | Limited display only | No | Low | Suitable only for simple outbound notifications |
| Microsoft Teams Workflows | Partial | Usually yes | Limited | Limited | Yes | Limited | Low to Moderate | Appropriate for lightweight automation, not core governed approvals |
| Power Automate | Yes | Usually yes | Partial | Limited to Moderate | Yes | Yes | Moderate | Useful for low-code orchestration, but not the strongest product-grade interaction model |
| Email-to-Teams | Yes, if enabled | Usually yes | No | No | No | No | Low | Notification-only option |

### 8.2 Capability Comparison Table

| Capability | Teams App + Bot Framework | Microsoft Graph App-only | Incoming Webhook | Microsoft Teams Workflows | Power Automate | Email-to-Teams |
|---|---|---|---|---|---|---|
| Channel notifications | Yes | Yes | Yes | Yes | Yes | Yes |
| Adaptive Cards | Yes | Limited | Limited | Yes | Yes | No |
| Inline approve/reject | Yes | No | No | Limited | Yes | No |
| Update/disable card after action | Yes | Limited | No | Limited | Moderate | No |
| 1:1 notifications | Yes | Limited | No | Limited | Yes | No |
| Create team/channel | With Graph support | Yes | No | No | Possible with additional setup | No |
| SSO / identity-aware action | Yes | Partial | No | Limited | Limited | No |
| RBAC-secure approval flow | Yes | No alone | No | Limited | Partial | No |
| Stale-card/version validation | Yes | No alone | No | Limited | Limited | No |
| Audit-grade callback handling | Yes | Partial | No | Limited | Moderate | No |
| Deep link to CARBON | Yes | Yes | Yes | Yes | Yes | Yes |
| Embed document review / tab experience | Yes | No | No | No | No | No |
| Document review support | Yes | Partial | No | Limited | Limited to Moderate | No |
| Enterprise suitability | Best | Partial | Low | Low to Moderate | Moderate | Low |

---

## 9. Formal Assessment of Integration Options

### 9.1 Teams App + Bot Framework

Teams App + Bot Framework represents the strongest and most strategically aligned option for CARBON’s proposed Teams integration.

#### Strengths

This option provides:

- proactive messaging into Teams channels  
- rich Adaptive Card support  
- inline approve and reject actions  
- direct callback handling  
- in-place card update and disable capability  
- support for identity-aware interaction  
- extensibility into 1:1, personal app, and tab scenarios  
- support for document review-oriented experiences  
- strong long-term product alignment  

#### Alignment with CARBON Requirements

This option most effectively supports:

- secure enterprise approval workflows  
- backend-controlled RBAC enforcement  
- stale-card validation at time of action  
- immutable audit traceability  
- rich approval context  
- validation alerting  
- future extensibility for broader Teams-based experiences  

#### Considerations

The principal consideration is deployment governance in customer-owned tenants. This model typically requires customer admin participation during onboarding.

#### Conclusion

**Assessment:** Best fit  
**Recommendation:** Use as the primary solution

### 9.2 Microsoft Graph App-only

Microsoft Graph App-only is highly valuable as an enabling platform capability, but it is not sufficient as the primary user interaction model.

#### Strengths

This option supports:

- provisioning operations  
- channel/team management operations  
- service-to-service integration  
- background operational tasks  
- administrative support capabilities  

#### Alignment with CARBON Requirements

Graph is useful for supporting the overall solution architecture, particularly in areas such as setup, provisioning, and management automation.

#### Limitations

It does not natively provide the full interaction model required for:

- inline approval decisions  
- controlled card lifecycle management  
- rich callback-owned workflow experience  
- document review interaction  

#### Conclusion

**Assessment:** Partially suitable  
**Recommendation:** Use only as a supporting technology

### 9.3 Incoming Webhook

Incoming Webhook is simple but materially underpowered for the proposed enterprise workflow.

#### Strengths

It supports:

- low-complexity channel notifications  
- basic outbound communication into Teams  

#### Alignment with CARBON Requirements

It may serve limited informational use cases, such as notifying a team that a section requires attention and linking users to CARBON.

#### Limitations

It does not adequately support:

- secure interaction  
- identity-aware decisions  
- RBAC validation  
- stale-card handling  
- lifecycle ownership of approval cards  
- audit-grade action processing  
- document review  

#### Conclusion

**Assessment:** Low suitability  
**Recommendation:** Not suitable for the core solution

### 9.4 Microsoft Teams Workflows

Microsoft Teams Workflows offers a low-code automation approach within Teams and can be valuable in simple process scenarios. However, it is not suitable as the primary architecture for CARBON’s disclosure approval workflow.

#### Strengths

It supports:

- basic automation  
- lightweight card-driven experiences  
- reminders and simple process triggers  
- low-code orchestration in Teams-centric scenarios  

#### Alignment with CARBON Requirements

Teams Workflows may support ancillary capabilities such as reminders, escalations, or lightweight supporting notifications.

#### Why It Is Not Suitable as the Primary Model

It is not sufficiently strong in the areas most critical to CARBON’s requirements:

1. **Enterprise approval control**  
   CARBON requires deterministic backend enforcement of identity, RBAC, workflow state, and content version at the moment of decision.

2. **Card lifecycle governance**  
   The requirement includes in-place card updates, stale-card handling, superseding prior cards, and inactivity controls. Teams Workflows does not provide the same degree of lifecycle control as a dedicated bot-based model.

3. **Stale-card and concurrency sensitivity**  
   The CARBON approval model depends on precise validation of section freshness and workflow state at action time. Teams Workflows is less well suited to this stringent, version-aware interaction pattern.

4. **Rich document review support**  
   CARBON requires not only notifications, but meaningful review support, including diff visibility and future potential for embedded review surfaces. Teams Workflows is not intended to be the primary review experience layer for such a product.

5. **Audit-sensitive enterprise interaction model**  
   CARBON requires an interaction pattern that supports strict traceability, strong backend ownership, and product-grade consistency. Teams Workflows is better suited to lightweight process automation than a governed approval platform.

6. **Cross-tenant governance still applies**  
   Even if Teams Workflows is used, enterprise tenant policies and customer admin controls remain relevant. It does not eliminate deployment governance considerations.

#### Conclusion

**Assessment:** Low to moderate suitability  
**Recommendation:** May be used for supporting automation only; not suitable as the core solution

### 9.5 Power Automate

Power Automate is more capable than Teams Workflows and can support broader automation across Microsoft services. It is a stronger low-code option, but it remains less suitable than a dedicated Teams App + Bot approach for CARBON’s product requirements.

#### Strengths

It supports:

- low-code orchestration  
- flow-based approval models  
- Teams notifications  
- broader connector ecosystem  
- cross-system process automation  

#### Alignment with CARBON Requirements

Power Automate may help with:

- supporting process automation  
- ancillary approvals  
- notifications and orchestration around CARBON events  
- backend-triggered integrations  

#### Limitations

It is still not the strongest fit for:

- product-grade control of interactive card lifecycle  
- precise stale-card validation behavior  
- highly customized enterprise interaction design  
- rich review context  
- strict product-owned approval UX consistency  

#### Conclusion

**Assessment:** Moderate suitability  
**Recommendation:** Suitable for supporting automation, but not ideal as the core interaction model

### 9.6 Email-to-Teams

Email-to-Teams is the least capable option under consideration.

#### Strengths

It can support:

- basic informational posting into a Teams channel  
- simple links to external systems  

#### Alignment with CARBON Requirements

Its relevance is limited to passive notification.

#### Limitations

It does not support:

- Adaptive Cards  
- inline approval actions  
- identity-aware callbacks  
- stale-card handling  
- card updates  
- audit-grade interaction  
- document review  

#### Conclusion

**Assessment:** Low suitability  
**Recommendation:** Not suitable for the proposed solution

### 9.7 Comparative Summary

The comparative analysis leads to the following conclusions:

- **Teams App + Bot Framework** is the only option that strongly satisfies the full set of enterprise requirements for CARBON’s Teams-based approval workflow.  
- **Microsoft Graph App-only** is valuable as a supporting technology, especially for provisioning and management, but not as the primary interaction surface.  
- **Incoming Webhook** and **Email-to-Teams** are effectively notification-only options and do not meet workflow, security, or audit requirements.  
- **Microsoft Teams Workflows** and **Power Automate** provide varying degrees of low-code automation support, but neither provides the same level of control, extensibility, and product alignment as a dedicated Teams app and bot.  

### 9.8 Recommended Best-Suited Option

#### Recommended Solution  
**Microsoft Teams App + Bot Framework, supported by Microsoft Graph for provisioning and channel-related operations where required**

#### Rationale

This approach is recommended because it best supports:

- secure and interactive approval actions  
- controlled card lifecycle behavior  
- version-aware stale-card validation  
- audit-grade transaction handling  
- diff-based review support  
- validation alerting  
- future extensibility into broader Teams experiences  

#### Management Positioning

This solution should be positioned as a **customer-admin-onboarded enterprise integration model** for customer-owned Teams tenants.

---

## 10. Proposed Solution Architecture

The proposed architecture is based on a clear separation of responsibilities between Microsoft Teams and IRIS CARBON.

### 10.1 Microsoft Teams as the Interaction Layer

Teams will serve as the operational user interface for:

- approval notifications  
- review interaction  
- validation alert visibility  
- user-triggered approval or rejection actions  
- deep-link navigation into CARBON  

### 10.2 IRIS CARBON as the System of Record

CARBON will remain responsible for:

- workflow state  
- authorization and RBAC enforcement  
- section and document versioning  
- stale-card validation  
- validation logic  
- diff computation  
- persistence of decisions  
- audit logging  

### 10.3 Core Solution Components

The high-level solution should include:

1. CARBON Workflow Service  
2. Teams Bot Service  
3. Entra ID Validation Layer  
4. CARBON Authorization and Policy Layer  
5. Audit Logging Service  
6. Diff Generation Service  
7. Validation Alert Service  
8. Microsoft Graph Support Layer  

---

## 11. Proposed End-to-End Operating Model

### 11.1 Approval Flow

1. A section transitions into `PENDING_APPROVAL`  
2. CARBON computes the approval context and change summary  
3. The Teams bot posts the approval card to the mapped organization channel  
4. The approver reviews the summary and takes action in Teams  
5. The action is submitted to CARBON  
6. CARBON validates:
   - identity  
   - authorization  
   - workflow state  
   - version freshness  
   - card validity  
7. If valid, CARBON applies the decision  
8. The audit service records the decision  
9. The Teams card is updated in place and actions are disabled  

### 11.2 Stale-Card Handling

1. A user acts on an earlier card  
2. CARBON detects state or version mismatch  
3. The action is rejected  
4. The card is marked stale or superseded  
5. A refreshed card may be issued where required  

### 11.3 Validation Alert Flow

1. A validation issue is detected  
2. CARBON evaluates deduplication and current issue state  
3. The Teams bot posts or updates the corresponding alert card  
4. Users take action in Teams or navigate to CARBON  
5. Once resolved, the card is updated to reflect the new status  

---

## 12. Business, Technical, and Compliance Benefits

### 12.1 Business Benefits

- accelerated approval turnaround  
- reduced context switching  
- improved productivity for approvers  
- improved visibility of changes and issues  
- faster triage of validation failures  
- more efficient disclosure cycle execution  

### 12.2 Technical Benefits

- centralized policy enforcement in CARBON  
- robust stale-card and version validation  
- identity-aware action handling  
- extensible Teams interaction model  
- support for future product evolution in Teams  

### 12.3 Compliance and Governance Benefits

- immutable audit trail  
- source-channel traceability  
- tamper-evident event verification  
- strong separation of interaction and policy enforcement  
- enterprise-ready organizational isolation  

---

## 13. Constraints, Assumptions, and Risks

### 13.1 Customer Admin Participation

Cross-tenant enterprise deployment will generally require customer administrator participation for tenant enablement and application availability.

### 13.2 Tenant Policy Variability

Customers may differ in:

- Teams app governance  
- bot enablement  
- consent policy  
- external user controls  
- organizational restrictions  

### 13.3 Rollout Complexity

Customer onboarding and operational readiness must be planned as part of implementation.

### 13.4 Functional Clarification Required

The rejection state transition must be formally clarified prior to design and build.

---

## 14. Final Recommendation

Following a full comparative review of the available Microsoft Teams integration approaches, the recommended course of action is:

**Proceed with a Microsoft Teams App + Bot Framework based integration for IRIS CARBON, using Microsoft Graph as a supporting capability for provisioning and channel-related operations where required.**

This option is the most appropriate because it provides the strongest fit across:

- enterprise workflow requirements  
- interaction quality  
- security and identity handling  
- stale-card prevention  
- auditability  
- document review support  
- long-term product extensibility  

The solution should be formally positioned as a **customer-admin-onboarded enterprise integration** for customer-owned Teams environments.

---

## 15. Appendices

### 15.1 Appendix A – Cross-Tenant Feasibility Table

| Option | Works in customer’s Teams tenant | Needs customer admin involvement | Supports secure approval workflow | Supports Document Review | Supports Adaptive Cards | Supports inline approve/reject callback | Enterprise suitability for requirement | Notes |
|---|---|---|---|---|---|---|---|---|
| Teams App + Bot Framework | Yes | Yes, typically required | Yes | Yes | Yes | Yes | Best | Most suitable for secure, governed, interactive enterprise workflow |
| Microsoft Graph App-only | Partial | Yes | Partial / No alone | Partial | Limited | No | Partial | Supporting technology only |
| Incoming Webhook | Yes, if enabled | Usually yes | No | No | Limited display only | No | Low | Simple notification option |
| Microsoft Teams Workflows | Partial | Usually yes | Limited | Limited | Yes | Limited | Low to Moderate | Useful for lightweight supporting automation |
| Power Automate | Yes | Usually yes | Partial | Limited to Moderate | Yes | Yes | Moderate | Stronger low-code option, but not ideal as core product interaction layer |
| Email-to-Teams | Yes, if enabled | Usually yes | No | No | No | No | Low | Notification-only option |

### 15.2 Appendix B – Capability Comparison Table

| Capability | Teams App + Bot Framework | Microsoft Graph App-only | Incoming Webhook | Microsoft Teams Workflows | Power Automate | Email-to-Teams |
|---|---|---|---|---|---|---|
| Channel notifications | Yes | Yes | Yes | Yes | Yes | Yes |
| Adaptive Cards | Yes | Limited | Limited | Yes | Yes | No |
| Inline approve/reject | Yes | No | No | Limited | Yes | No |
| Update/disable card after action | Yes | Limited | No | Limited | Moderate | No |
| 1:1 notifications | Yes | Limited | No | Limited | Yes | No |
| Create team/channel | With Graph support | Yes | No | No | Possible with additional setup | No |
| SSO / identity-aware action | Yes | Partial | No | Limited | Limited | No |
| RBAC-secure approval flow | Yes | No alone | No | Limited | Partial | No |
| Stale-card/version validation | Yes | No alone | No | Limited | Limited | No |
| Audit-grade callback handling | Yes | Partial | No | Limited | Moderate | No |
| Deep link to CARBON | Yes | Yes | Yes | Yes | Yes | Yes |
| Embed document review / tab experience | Yes | No | No | No | No | No |
| Document review support | Yes | Partial | No | Limited | Limited to Moderate | No |
| Enterprise suitability | Best | Partial | Low | Low to Moderate | Moderate | Low |
