# Teams Integration Options

## Table of Contents
1. [Background Comparison Table](#background-comparison-table)
2. [Option 1: Single Teams App per-Tenant](#option-1-single-teams-app-per-tenant)
   - [Problem Statement](#problem-statement)
   - [Requirements](#requirements)
   - [Architecture](#architecture)
   - [Implementation Plan](#implementation-plan)
   - [Challenges](#challenges)
   - [Pros/Cons](#proscons)
3. [Option 2: Multi-Tenant App](#option-2-multi-tenant-app)
   - [Problem Statement](#problem-statement-1)
   - [Requirements](#requirements-1)
   - [Architecture](#architecture-1)
   - [Implementation Plan](#implementation-plan-1)
   - [Challenges](#challenges-1)
   - [Pros/Cons](#proscons-1)
   - [Prerequisites](#prerequisites)
   - [Admin Consent Flow](#admin-consent-flow)
4. [No-Admin-Consent Approaches](#no-admin-consent-approaches)
   - [Incoming Webhook](#incoming-webhook)
   - [Power Automate](#power-automate)
   - [Email Bridge](#email-bridge)
5. [REST API Section](#rest-api-section)
6. [PDF Export Instructions](#pdf-export-instructions)

---

## Background Comparison Table
| Feature                        | Web-only                  | Teams-integrated           |
|--------------------------------|--------------------------|-----------------------------|
| Authentication                  | OAuth2                   | OAuth2                      |
| User Interaction                | Browser UI               | Teams UI                    |
| Notification Handling           | Web Notifications         | Teams Notifications          |

## Option 1: Single Teams App per-Tenant
### Problem Statement
To provide seamless integration for a single organization into Microsoft Teams.

### Requirements
- Microsoft Azure Subscription
- Teams Administrator

### Architecture
