# Okta Identity Lifecycle Management — Joiner Mover Leaver (JML) with BambooHR and SCIM Provisioning

> A hands-on implementation of a complete enterprise identity lifecycle using Okta as the Identity Provider, BambooHR as the authoritative HR source, and a Sterling Bank SCIM application as the downstream resource server. This lab demonstrates automated user provisioning, attribute synchronisation, group-based access control, and deprovisioning across all three phases of the JML lifecycle.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture and Data Flow](#architecture-and-data-flow)
- [JML Lifecycle Flow](#jml-lifecycle-flow)
- [Prerequisites](#prerequisites)
- [Lab Steps](#lab-steps)
  - [Step 1 — Connect BambooHR to Okta (HR Integration)](#step-1--connect-bamboohr-to-okta-hr-integration)
  - [Step 2 — Configure Import and Lifecycle Sourcing Rules](#step-2--configure-import-and-lifecycle-sourcing-rules)
  - [Step 3 — Import Users from BambooHR](#step-3--import-users-from-bamboohr)
  - [Step 4 — Create a Dynamic Group in Okta](#step-4--create-a-dynamic-group-in-okta)
  - [Step 5 — Get a Free SCIM Test Endpoint (scim.dev)](#step-5--get-a-free-scim-test-endpoint-scimdev)
  - [Step 6 — Add the SCIM 2.0 Test App in Okta](#step-6--add-the-scim-20-test-app-in-okta)
  - [Step 7 — Connect Okta to scim.dev and Enable Provisioning](#step-7--connect-okta-to-scimdev-and-enable-provisioning)
  - [Step 8 — Assign the Application to the Dynamic Group](#step-8--assign-the-application-to-the-dynamic-group)
  - [Step 9 — Test the Joiner Workflow](#step-9--test-the-joiner-workflow)
  - [Step 10 — Test the Mover Workflow](#step-10--test-the-mover-workflow)
  - [Step 11 — Test the Leaver Workflow](#step-11--test-the-leaver-workflow)
- [Screenshots](#screenshots)
- [Best Practices Summary](#best-practices-summary)
- [JML Workflow Reference](#jml-workflow-reference)
- [Key Concepts Reference](#key-concepts-reference)
- [Resources](#resources)

---

## Overview

This lab implements a **Joiner Mover Leaver (JML) identity lifecycle** — the foundational pattern of enterprise IAM — using three systems in an automated provisioning chain:

| System | Role | Responsibility |
|---|---|---|
| **BambooHR** | Authoritative Source (HR System of Record) | Owns employee identity data — creates, updates, and terminates employees |
| **Okta** | Identity Provider (IdP) / Identity Broker | Receives identity events from BambooHR, enforces group membership rules, and pushes access to downstream applications |
| **Sterling Bank Application** | Resource Server (Service Provider) | Receives provisioned users from Okta via SCIM; grants application access based on group membership and role attributes |

**The core principle:** Identity data flows in one direction — from HR (the authoritative source) downstream to applications. No application creates or owns identity data. HR is always the source of truth.

---

## Architecture and Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          IDENTITY LIFECYCLE ARCHITECTURE                    │
└─────────────────────────────────────────────────────────────────────────────┘

                    AUTHORITATIVE            IDENTITY              RESOURCE
                       SOURCE                BROKER                SERVER
                    ┌───────────┐         ┌──────────┐          ┌───────────┐
                    │           │  Import  │          │   SCIM   │           │
                    │ BambooHR  │─────────►│   Okta   │─────────►│SterlingBank│
                    │           │ (hourly  │   (IdP)  │  Push    │    App     │
                    │ HR System │  sync)   │          │          │            │
                    └───────────┘         └──────────┘          └─────────────┘
                          │                    │                      │
                    Employees,           Dynamic Groups,         User Accounts,
                    Departments,         Attribute Rules,        Role Attributes,
                    Job Titles           Profile Mapping         Access Control


┌─────────────────────────────────────────────────────────────────────────────┐
│                          DETAILED DATA FLOW                                 │
└─────────────────────────────────────────────────────────────────────────────┘

BambooHR                    Okta                           SterlingBank
──────────                  ──────────────────────────     ──────────────────
Employee record             ① Import user profile          ① SCIM creates user
  firstName                     ↓                              account
  lastName             ② Evaluate group rules:            ② Attribute mapping
  department               IF dept = "IT"                      applied:
  title                    AND title = "Senior Dev"            title → role
  email                    → add to SterlingBank IT Admin      ③ User accesses app
    │                          Group                           with correct
    │                          ↓                              permissions
    └──────────────────► ③ Group assigned to SterlingBank
                             application
                             ↓
                         ④ SCIM push triggers
                             user creation
                             in SterlingBank
```

---

## JML Lifecycle Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     JOINER ─── MOVER ─── LEAVER                            │
└─────────────────────────────────────────────────────────────────────────────┘

  JOINER                    MOVER                     LEAVER
  ──────                    ─────                     ──────
  New employee              Employee profile          Employee terminated
  created in                updated in                or deleted in
  BambooHR                  BambooHR                  BambooHR
       │                         │                         │
       ▼                         ▼                         ▼
  Okta imports             Okta imports              Okta imports
  user on next             delta on next             user removal
  scheduled sync           scheduled sync            on next sync
       │                         │                         │
       ▼                         ▼                         ▼
  Group rules              Group rules               User deactivated
  evaluated —              re-evaluated —            in Okta
  user added to            user moved                     │
  matching groups          between groups                 ▼
       │                   if criteria              User deactivated
       ▼                   changed                  in SterlingBank via
  SCIM pushes                   │                   SCIM deprovision
  new user to                   ▼                        │
  SterlingBank                  SCIM pushes               Session invalidated
       │                   updated attrs             Access removed
       ▼                   to SterlingBank
  User can log                  │
  into SterlingBank             User sees
  via Okta SSO             updated profile
                           and correct role
```

---

## Prerequisites

- **Okta tenant** with Super Administrator access
- **BambooHR tenant** with API access enabled and test employees populated
- **SterlingBank application** (or equivalent custom app) with:
  - A SCIM 2.0 endpoint (HTTPS required)
  - Bearer token or HTTP header authentication for SCIM
  - SAML SSO endpoint configured
  - A tunnelling tool (e.g. ngrok) if running locally, to expose SCIM endpoint over HTTPS
- BambooHR subdomain name (used during app integration setup)
- At least one test employee in BambooHR with `department = IT` and `title = Senior Developer` to trigger the dynamic group rule

---

## Lab Steps

---

### Step 1 — Connect BambooHR to Okta (HR Integration)

**Goal:** Register BambooHR as an application integration in Okta and authenticate the API connection so Okta can import employee data.

1. Sign in to your **Okta Admin Console**.
2. Navigate to **Applications** → **Applications** → **Browse App Catalog**.
3. Search for **BambooHR** and select the integration.
4. Click **Add Integration**.
5. On the **General Settings** tab, enter your BambooHR **subdomain name** (the prefix of your BambooHR URL — e.g. if your URL is `mintis.bamboohr.com`, your subdomain is `mintis`).
6. Click **Done**.
7. Navigate to the **Provisioning** tab of the newly added BambooHR integration.
8. Click **Configure API Integration**.
9. Click **Authenticate with BambooHR** — this will redirect to BambooHR to complete OAuth authorisation.
10. Confirm the message: *BambooHR API is authenticated.*
11. Click **Save**.

> **What just happened:** Okta now has an authenticated API connection to your BambooHR tenant. It can read employee records, department data, job titles, and group memberships from BambooHR on a scheduled basis.

#### Screenshot
<!-- Add your screenshot here -->
![Step 1 - BambooHR App Integration and API Authentication](_screenshots/step1-bamboohr-authenticated.png)

---

### Step 2 — Configure Import and Lifecycle Sourcing Rules

**Goal:** Define how Okta imports users from BambooHR and how HR-driven lifecycle events (deactivation, reactivation) flow through to Okta.

1. On the BambooHR integration **Provisioning** tab, click **Edit** under **To Okta** settings.
2. Configure the **General** section:
   - **Schedule import:** Set to **Every hour** (BambooHR does not support real-time attribute sync — scheduled import is required)
3. Configure **Profile & Lifecycle Sourcing**:
   - Enable **Allow BambooHR to source Okta users** — this makes BambooHR the authoritative source for user profiles
   - **When a user is deactivated in BambooHR:** Select **Deactivate the user in Okta**
   - **When a user is reactivated in BambooHR:** Select **Reactivate suspended Okta users**
4. Click **Save**.

> **Authoritative source principle:** By enabling BambooHR as the profile source, Okta will not allow manual overrides to attributes that BambooHR owns. Changes to an employee's name, department, or title must be made in BambooHR — they will flow to Okta automatically on the next import cycle. This enforces data integrity and prevents identity drift.

#### Screenshot
<!-- Add your screenshot here -->
![Step 2 - Import Schedule and Lifecycle Sourcing Configuration](_screenshots/step2-import-lifecycle-config.png)

---

### Step 3 — Import Users from BambooHR

**Goal:** Trigger an immediate import to pull current BambooHR employees into Okta without waiting for the hourly schedule.

1. On the BambooHR integration page, click the **Import** tab.
2. Click **Import Now** to trigger a full import.
3. Wait for the import to complete — import duration depends on the number of employees and groups in BambooHR.
4. Review the import summary:
   - New users found
   - Existing users updated
   - Groups detected
5. Select the users to import and set their status to **Managed** (Okta will manage their lifecycle).
6. Click **Confirm Assignments**.
7. Navigate to **Directory** → **Groups** to review imported BambooHR groups:
   - BambooHR-sourced groups will show **Managed by BambooHR** — membership cannot be edited manually in Okta
   - These groups reflect org structure from BambooHR; you will create your own Okta groups for application access control (Step 4)

> **Key distinction:** BambooHR groups reflect org chart structure. Okta dynamic groups (created in Step 4) are what you use to control application access. Do not rely on BambooHR groups for application assignment — build Okta-native groups with expression rules instead.

#### Screenshot
<!-- Add your screenshot here -->
![Step 3 - Import Results and User Confirmation](_screenshots/step3-import-results.png)

---

### Step 4 — Create a Dynamic Group in Okta

**Goal:** Create an Okta group with an expression-based membership rule that automatically includes users meeting specific criteria — in this case, IT department employees with the Senior Developer title.

1. Navigate to **Directory** → **Groups**.
2. Click **Add Group**.
3. Enter a descriptive name: `SterlingBank IT Admin`
4. Click **Save**.
5. Open the newly created group and click **Manage Rules**.
6. Click **Add Rule**.
7. Configure the rule:
   - **Name:** `IT Senior Developer Rule`
   - **IF:** Use the Okta Expression Language to define the condition:
     ```
     user.department == "IT" AND user.title == "Senior Developer"
     ```
   - **THEN assign to:** `SterlingBank IT Admin`
8. Click **Save Rule**.
9. Click **Activate** to enable the rule.
10. Return to the group — Okta will evaluate the rule against all current users and automatically add matching members.
11. Verify that the expected test user (e.g. Hillman Everest, department = IT, title = Senior Developer) appears as a member.

> **Why dynamic groups?** Manual group assignment does not scale and creates ongoing administrative overhead. A dynamic group with an expression rule automatically adds and removes users as their attributes change in BambooHR — no manual intervention required. When Hillman moves from IT to Finance, the rule evaluates on next import and removes him from `SterlingBank IT Admin` automatically.

```
Dynamic Group Rule Logic
────────────────────────
BambooHR Employee Record
    │
    ▼
Okta Profile
    │
    ▼
Rule Evaluation:
    IF user.department == "IT"
    AND user.title == "Senior Developer"
    ─────────────────────────────────────
    TRUE  → Add to "SterlingBank IT Admin"
    FALSE → Not assigned / removed if previously assigned
```

#### Screenshot
<!-- Add your screenshot here -->
![Step 4 - Dynamic Group Rule Configuration and Membership Preview](_screenshots/step4-dynamic-group-rule.png)

---

### Step 5 — Get a Free SCIM Test Endpoint (scim.dev)

**Goal:** Provision a public, authenticated SCIM 2.0 sandbox that Okta can push provisioning events to — without building or hosting any server infrastructure.

> **Why scim.dev instead of a custom server?**
> Okta is a SCIM *client* — it sends outbound HTTP requests whenever a user is created, updated, or deactivated. To observe and validate those requests, you need a SCIM-compliant endpoint that can receive them. `scim.dev` is an online mock SCIM sandbox recommended by Okta. It requires no code, no Docker, no ngrok, and no local server — you get a fully functional, authenticated SCIM 2.0 endpoint in under a minute.

```
What scim.dev provides
───────────────────────
Public SCIM 2.0 base URL: https://api.scim.dev/scim/v2
Authentication:           HTTP Header (Bearer token — your API key)
Endpoints available:      /Users   /Groups
HTTP log viewer:          Real-time display of every request Okta sends
User inspector:           View created/updated/deactivated user records
Retention:                Playground persists for your session
```

**Steps:**

1. Open a new browser tab and navigate to [https://scim.dev](https://scim.dev).
2. Click **Get an API Key**, accept the terms, and click **Access My Playground**.
3. On the playground page, locate and **copy your API Key** — you will paste this into Okta in Step 6.
4. Note your SCIM base URL — it is fixed for all `scim.dev` users:
   ```
   https://api.scim.dev/scim/v2
   ```
5. Keep this browser tab open alongside your Okta Admin Console throughout the lab — you will return here to observe live provisioning events.

#### Screenshot
<!-- Add your screenshot here -->
![Step 5 - scim.dev Playground with API Key](_screenshots/step5-scimdev-api-key.png)

---

### Step 6 — Add the SCIM 2.0 Test App in Okta

**Goal:** Register Okta's built-in SCIM 2.0 catalog test application — a pre-configured integration designed specifically for validating SCIM provisioning workflows.

> **Why use the catalog test app rather than a custom integration?**
> The `SCIM 2.0 Test App (Header Auth)` is an Okta-maintained integration built to exercise the full SCIM provisioning surface — create, update, deactivate, and group push. It removes all SAML configuration overhead so the lab stays focused on provisioning mechanics, which is the core learning objective here.

1. In the Okta Admin Console, navigate to **Applications** → **Applications**.
2. Click **Browse App Catalog**.
3. Search for `SCIM 2.0 Test App (Header Auth)`.
4. Select the result and click **Add Integration**.
5. Enter a descriptive name: `SterlingBank SCIM Practice App`
6. Click **Next**.
7. On the **Sign-On Options** tab, leave the default settings.
8. Click **Done**.

> The application is now registered in Okta. It has no SCIM endpoint connected yet — that is configured in Step 7.

#### Screenshot
<!-- Add your screenshot here -->
![Step 6 - SCIM 2.0 Test App Added from Okta Catalog](_screenshots/step6-scim-test-app-added.png)

---

### Step 7 — Connect Okta to scim.dev and Enable Provisioning

**Goal:** Point Okta's SCIM client at the `scim.dev` endpoint, authenticate the connection, and enable all three JML provisioning actions.

#### 7a — Configure the SCIM Connection

1. On the `SterlingBank SCIM Practice App` page, click the **Provisioning** tab.
2. Click **Configure API Integration**.
3. Check **Enable API Integration**.
4. Enter the following connection details:

   | Field | Value |
   |---|---|
   | **SCIM connector base URL** | `https://api.scim.dev/scim/v2` |
   | **Unique identifier field for users** | `userName` |
   | **Authentication Mode** | `HTTP Header` |
   | **Authorization** | Paste the API Key copied from `scim.dev` in Step 5 |

5. Click **Test API Credentials** — a green checkmark confirms the connection is authenticated.
6. Click **Save**.

```
SCIM Connection — What Okta Will Send
───────────────────────────────────────
Okta (SCIM Client)
    │
    │  HTTPS requests to https://api.scim.dev/scim/v2
    │  Authorization: Bearer <your-scim.dev-api-key>
    │
    ▼
scim.dev SCIM Endpoint
    │
    ├── POST   /Users              → Create user (Joiner)
    ├── PATCH  /Users/{id}         → Update attributes (Mover)
    ├── PATCH  /Users/{id}         → Deactivate user (Leaver)
    │         { "active": false }
    └── GET    /Users              → Okta reconciliation checks
```

> **HTTPS is mandatory for SCIM endpoints in production.** SCIM endpoints are privileged — they can create, update, and delete user accounts. `scim.dev` enforces HTTPS natively. When building your own SCIM server in a production environment, ensure it is behind a valid TLS certificate at all times.

#### Screenshot
<!-- Add your screenshot here -->
![Step 7a - SCIM API Integration Configured and Tested](_screenshots/step7a-scim-api-configured.png)

#### 7b — Enable Provisioning Actions (To App)

7. On the **Provisioning** tab, click **To App** in the left settings panel.
8. Click **Edit** and enable the following actions:

   | Action | JML Phase | What It Does |
   |---|---|---|
   | ✅ **Create Users** | **Joiner** | Sends `POST /Users` to scim.dev when a user is assigned to the app |
   | ✅ **Update User Attributes** | **Mover** | Sends `PATCH /Users/{id}` when a user's profile is updated in Okta |
   | ✅ **Deactivate Users** | **Leaver** | Sends `PATCH /Users/{id}` with `"active": false` when a user is unassigned or deactivated |

9. Leave **Sync Password** unchecked — authentication is handled separately via Okta SSO; passwords are not provisioned via SCIM.
10. Click **Save**.

> **Why exclude import from scim.dev?** `scim.dev` is a test sink — a destination that receives provisioning events. It is not an authoritative identity source. Enabling import from it would create circular data flow, with a mock sandbox overwriting real BambooHR identity data in Okta. The data flow must remain unidirectional: BambooHR → Okta → scim.dev.

#### Screenshot
<!-- Add your screenshot here -->
![Step 7b - Provisioning Actions Enabled (Create, Update, Deactivate)](_screenshots/step7b-provisioning-actions.png)

---

### Step 8 — Assign the Application to the Dynamic Group

**Goal:** Connect the `SterlingBank IT Admin` dynamic group to the SCIM test application so that all group members are automatically provisioned into the scim.dev endpoint.

1. On the `SterlingBank SCIM Practice App` page, navigate to the **Assignments** tab.
2. Click **Assign** → **Assign to Groups**.
3. Search for and select `SterlingBank IT Admin`.
4. Review the attribute mappings presented — accept defaults for now.
5. Click **Save and Go Back** → **Done**.
6. Verify that `SterlingBank IT Admin` appears in the **Assignments** tab.
7. Navigate to **Directory** → **People** and locate your test user (Hillman Everest).
8. Confirm that `SterlingBank SCIM Practice App` appears in the user's **Applications** tab — this confirms the provisioning chain from BambooHR through the dynamic group to the SCIM endpoint is fully active.

> **Group-based assignment is the correct production pattern.** Assigning applications directly to individual users creates an unmanageable access model at scale and requires manual intervention for every joiner and leaver. Group-based assignment means access is granted and revoked automatically as users enter and leave the dynamic group — driven entirely by BambooHR attribute changes.

```
Complete Provisioning Chain (confirmed at end of Step 8)
─────────────────────────────────────────────────────────
BambooHR employee record
    │  (hourly import)
    ▼
Okta user profile
    │  (dynamic group rule: dept=IT AND title=Senior Developer)
    ▼
SterlingBank IT Admin group
    │  (group assigned to app)
    ▼
SterlingBank SCIM Practice App
    │  (SCIM push to scim.dev)
    ▼
scim.dev /Users endpoint
    └── User account created / updated / deactivated
```

#### Screenshot
<!-- Add your screenshot here -->
![Step 8 - App Assigned to Dynamic Group; User Provisioning Chain Active](_screenshots/step8-app-group-assigned.png)

---

### Step 9 — Test the Joiner Workflow

**Goal:** Verify that a new employee created in BambooHR flows through Okta's dynamic group rule and triggers a `POST /Users` SCIM provisioning event, visible in scim.dev.

```
Joiner Signal Flow
───────────────────
BambooHR: new employee (dept=IT, title=Senior Developer)
    ↓  hourly import (or Import Now)
Okta: user created → dynamic rule matches → added to SterlingBank IT Admin
    ↓  group assigned to app → provisioning triggered
scim.dev: POST /Users received → user record created
```

#### 9a — Create a New Employee in BambooHR

1. In BambooHR, create a new employee with:
   - **Department:** `IT`
   - **Job Title:** `Senior Developer`
   - Complete all required fields (first name, last name, email)
2. Save the new employee record.

#### 9b — Trigger Import in Okta

3. In Okta, navigate to the BambooHR integration → **Import** tab.
4. Click **Import Now** to pull the new employee without waiting for the hourly schedule.
5. Review the import summary — confirm the new user is detected.
6. Confirm the import and set the user to **Managed**.

#### 9c — Verify Group Membership and Application Assignment

7. Navigate to **Directory** → **Groups** → `SterlingBank IT Admin`.
8. Confirm the new user has been automatically added by the dynamic group rule.
9. Navigate to the user's profile → **Applications** tab.
10. Confirm `SterlingBank SCIM Practice App` is listed as an assigned application.

#### 9d — Observe the SCIM Event in scim.dev

11. Switch to the `scim.dev` browser tab.
12. Navigate to **Logs** → **HTTP Logs**.
13. Confirm a `POST /Users` request was received from Okta. The payload will resemble:

    ```json
    {
      "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
      "userName": "hillman.everest@company.com",
      "name": {
        "givenName": "Hillman",
        "familyName": "Everest"
      },
      "emails": [{ "value": "hillman.everest@company.com", "primary": true }],
      "active": true
    }
    ```

14. Navigate to **Users** in scim.dev and confirm the user record has been created with the correct attributes.

> **What you are observing:** This is the exact JSON payload Okta sends to any SCIM-compliant application — including production systems — when a new user is provisioned. The same request that lands in scim.dev would land in a real enterprise application's SCIM endpoint in production.

#### Screenshots — Joiner Workflow
<!-- Add your screenshot here -->
![Step 9a - New Employee Created in BambooHR](_screenshots/step9a-new-employee-bamboohr.png)

<!-- Add your screenshot here -->
![Step 9b - Okta Import Detects New User](_screenshots/step9b-import-new-user.png)

<!-- Add your screenshot here -->
![Step 9c - User Added to Dynamic Group, App Assigned](_screenshots/step9c-group-membership-confirmed.png)

<!-- Add your screenshot here -->
![Step 9d - scim.dev HTTP Log: POST /Users Received](_screenshots/step9d-scimdev-post-users.png)

<!-- Add your screenshot here -->
![Step 9d - scim.dev User Record Created](_screenshots/step9d-scimdev-user-created.png)

---

### Step 10 — Test the Mover Workflow

**Goal:** Verify that a profile change in BambooHR propagates through Okta and triggers a `PATCH /Users/{id}` SCIM update event, observable in scim.dev.

```
Mover Signal Flow
──────────────────
BambooHR: employee profile updated (e.g. firstName changed)
    ↓  hourly import (or Import Now)
Okta: existing user profile updated in directory
    ↓  Update User Attributes provisioning action fires
scim.dev: PATCH /Users/{id} received → user record updated
```

#### 10a — Update the Employee in BambooHR

1. In BambooHR, locate the test user's profile.
2. Update an attribute — for example, change the employee's **First Name** (e.g. from `Hillman` to `Mountain Man`).
3. Save the updated record.

#### 10b — Trigger Import in Okta

4. In Okta, navigate to the BambooHR integration → **Import** tab.
5. Click **Import Now** to trigger a full import.
6. Wait for completion — the import summary should show **one existing user updated**.

#### 10c — Verify Profile Update in Okta

7. Navigate to **Directory** → **People** and locate the updated user.
8. Confirm the first name has been updated in the Okta user profile to reflect the BambooHR change.

#### 10d — Observe the SCIM Update Event in scim.dev

9. Switch to the `scim.dev` browser tab.
10. Navigate to **Logs** → **HTTP Logs**.
11. Confirm a `PATCH /Users/{id}` request was received from Okta. The payload will resemble:

    ```json
    {
      "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
      "Operations": [
        {
          "op": "Replace",
          "path": "name.givenName",
          "value": "Mountain Man"
        }
      ]
    }
    ```

12. Navigate to **Users** in scim.dev and confirm the user record reflects the updated first name.

> **Near-real-time attribute sync:** The one-hour BambooHR import schedule means attribute changes propagate within at most one hour in this lab setup. In production environments where tighter sync is required, evaluate whether your HR system supports webhook-based push notifications to Okta — which would trigger imports immediately on record changes rather than on a fixed schedule.

#### Screenshots — Mover Workflow
<!-- Add your screenshot here -->
![Step 10a - Employee Profile Updated in BambooHR](_screenshots/step10a-profile-updated-bamboohr.png)

<!-- Add your screenshot here -->
![Step 10b - Okta Import Shows User Updated](_screenshots/step10b-okta-profile-updated.png)

<!-- Add your screenshot here -->
![Step 10c - scim.dev HTTP Log: PATCH /Users Received](_screenshots/step10c-scimdev-patch-users.png)

---

### Step 11 — Test the Leaver Workflow

**Goal:** Verify that deleting or terminating an employee in BambooHR results in their Okta account being deactivated and a `PATCH /Users/{id}` deactivation event being sent to scim.dev.

```
Leaver Signal Flow
───────────────────
BambooHR: employee deleted or terminated
    ↓  hourly import (or Import Now)
Okta: user deactivated → removed from SterlingBank IT Admin group
    ↓  Deactivate Users provisioning action fires
scim.dev: PATCH /Users/{id} with "active": false received
          → user account disabled
```

#### 11a — Delete or Terminate the Employee in BambooHR

1. In BambooHR, locate the test user's record.
2. Delete or terminate the employee.
3. Save the change.

#### 11b — Trigger Import in Okta

4. In Okta, navigate to the BambooHR integration → **Import** tab.
5. Click **Import Now** to trigger a full import.
6. The import summary should show **one user removed**.

#### 11c — Verify Deactivation in Okta

7. Navigate to **Directory** → **People**.
8. Locate the user — their status should now show as **Deactivated**.
9. Confirm they have been automatically removed from the `SterlingBank IT Admin` group.
10. Confirm `SterlingBank SCIM Practice App` no longer appears in their **Applications** tab.

#### 11d — Observe the SCIM Deactivation Event in scim.dev

11. Switch to the `scim.dev` browser tab.
12. Navigate to **Logs** → **HTTP Logs**.
13. Confirm a `PATCH /Users/{id}` deactivation request was received from Okta. The payload will resemble:

    ```json
    {
      "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
      "Operations": [
        {
          "op": "Replace",
          "path": "active",
          "value": false
        }
      ]
    }
    ```

14. Navigate to **Users** in scim.dev and confirm the user's `active` field is now `false`.

#### 11e — Verify Okta Login Is Denied

15. Open a private/incognito browser window.
16. Attempt to log in to the Okta End User Dashboard as the deactivated user.
17. Confirm the login is denied — the account is deactivated and no session can be established.

```
Leaver Workflow Verification Checklist
────────────────────────────────────────
[ ] Employee deleted/terminated in BambooHR
[ ] Okta import reflects removal — import summary shows "one user removed"
[ ] User status = Deactivated in Okta directory
[ ] User removed from SterlingBank IT Admin group automatically
[ ] SterlingBank SCIM Practice App removed from user's Applications tab
[ ] scim.dev HTTP Log shows PATCH /Users/{id} with "active": false
[ ] scim.dev user record shows active = false
[ ] Okta login denied for deactivated user account
```

#### Screenshots — Leaver Workflow
<!-- Add your screenshot here -->
![Step 11a - Employee Deleted or Terminated in BambooHR](_screenshots/step11a-employee-deleted-bamboohr.png)

<!-- Add your screenshot here -->
![Step 11b - Okta Import Shows One User Removed](_screenshots/step11b-okta-user-removed.png)

<!-- Add your screenshot here -->
![Step 11c - User Deactivated in Okta Directory](_screenshots/step11c-user-deactivated-okta.png)

<!-- Add your screenshot here -->
![Step 11d - scim.dev HTTP Log: PATCH /Users active=false](_screenshots/step11d-scimdev-deactivation-patch.png)

<!-- Add your screenshot here -->
![Step 11d - scim.dev User Record: active=false](_screenshots/step11d-scimdev-user-inactive.png)


---

## Screenshots

Place your lab screenshots in the `_screenshots/` folder in this repository. Suggested naming:

```
_screenshots/
│
│   # HR Integration (Steps 1–3)
├── step1-bamboohr-authenticated.png          ← BambooHR API authentication confirmed in Okta
├── step2-import-lifecycle-config.png         ← Import schedule + lifecycle sourcing settings
├── step3-import-results.png                  ← Import summary showing users and groups found
│
│   # Dynamic Group (Step 4)
├── step4-dynamic-group-rule.png              ← Group rule expression and membership preview
│
│   # SCIM Test Endpoint Setup (Step 5)
├── step5-scimdev-api-key.png                 ← scim.dev playground with API key visible
│
│   # Okta SCIM Test App Registration (Step 6)
├── step6-scim-test-app-added.png             ← SCIM 2.0 Test App added from Okta catalog
│
│   # SCIM Connection and Provisioning Actions (Step 7)
├── step7a-scim-api-configured.png            ← SCIM connector settings + green credential test
├── step7b-provisioning-actions.png           ← Create / Update / Deactivate actions enabled
│
│   # Group-to-App Assignment (Step 8)
├── step8-app-group-assigned.png              ← SterlingBank IT Admin assigned to SCIM app
│
│   # Joiner Workflow (Step 9)
├── step9a-new-employee-bamboohr.png          ← New employee record created in BambooHR
├── step9b-import-new-user.png                ← Okta import detects and confirms new user
├── step9c-group-membership-confirmed.png     ← User auto-added to group; app assigned
├── step9d-scimdev-post-users.png             ← scim.dev HTTP log: POST /Users payload
├── step9d-scimdev-user-created.png           ← scim.dev Users tab: new user record visible
│
│   # Mover Workflow (Step 10)
├── step10a-profile-updated-bamboohr.png      ← Employee attribute updated in BambooHR
├── step10b-okta-profile-updated.png          ← Okta directory reflects updated attribute
├── step10c-scimdev-patch-users.png           ← scim.dev HTTP log: PATCH /Users payload
│
│   # Leaver Workflow (Step 11)
├── step11a-employee-deleted-bamboohr.png     ← Employee deleted/terminated in BambooHR
├── step11b-okta-user-removed.png             ← Okta import summary: one user removed
├── step11c-user-deactivated-okta.png         ← Okta directory: user status = Deactivated
├── step11d-scimdev-deactivation-patch.png    ← scim.dev HTTP log: PATCH active=false payload
├── step11d-scimdev-user-inactive.png         ← scim.dev Users tab: active=false confirmed
```

---

## Best Practices Summary

| # | Practice |
|---|---|
| 1 | **HR is always the authoritative source** — never create or modify identity records in Okta or downstream apps; all changes originate in HR |
| 2 | **Use dynamic groups with expression rules** rather than manual group assignment — scales automatically as attributes change |
| 3 | **SCIM endpoints must use HTTPS** — SCIM can create, update, and delete user accounts; never transmit over plain HTTP |
| 4 | **Exclude import from resource servers** — users should never flow back from an application into Okta; data flow is unidirectional |
| 5 | **Use email as the unique user identifier for SCIM** — email is stable, unique, and present across all systems in the chain |
| 6 | **Assign applications to groups, not individual users** — individual assignment creates an unmanageable model at scale |
| 7 | **Do not sync passwords via SCIM** — authentication is handled by SAML SSO; password sync creates credential sprawl |
| 8 | **Test all three JML phases** — a provisioning integration that only works for joiners but fails silently on leavers is a security risk |
| 9 | **Verify deprovisioning in the resource server**, not just in Okta — confirm the downstream application account is actually disabled, not just Okta |
| 10 | **Document your import schedule** — BambooHR's hourly sync means access is not revoked instantaSterlingusly; factor this into your offboarding SLA |

---

## JML Workflow Reference

| Phase | Trigger Event (BambooHR) | Okta Action | SterlingBank Action |
|---|---|---|---|
| **Joiner** | New employee created | User imported → group rule adds to `SterlingBank IT Admin` → app assigned | SCIM creates user account |
| **Mover** | Employee profile updated (name, title, department) | Attribute updated in Okta profile | SCIM PATCH updates user attributes in SterlingBank |
| **Mover (role change)** | Department or title changes so rule no longer matches | User removed from `SterlingBank IT Admin` group | SCIM deactivates user in SterlingBank |
| **Leaver** | Employee terminated or deleted | User deactivated in Okta → removed from all groups | SCIM deactivates user account; session invalidated |

---

## Key Concepts Reference

| Term | Description |
|---|---|
| **JML (Joiner Mover Leaver)** | The three lifecycle phases every employee identity passes through — joining the org, changing roles, and leaving |
| **Authoritative Source** | The system of record for identity data; all other systems receive data from it, never write back to it |
| **SCIM (System for Cross-domain Identity Management)** | A REST API standard for automating user provisioning and deprovisioning between identity systems and applications |
| **Dynamic Group** | An Okta group whose membership is automatically managed by an expression rule rather than manual assignment |
| **Okta Expression Language** | A syntax for writing attribute-based rules in Okta (e.g. `user.department == "IT"`) |
| **Provisioning** | The automated creation of user accounts and access in downstream systems |
| **Deprovisioning** | The automated removal or deactivation of user accounts when access is no longer needed |
| **Profile Sourcing** | Designating an application (e.g. BambooHR) as the authoritative source for a user's profile attributes in Okta |
| **SAML** | Standard for SSO authentication — handles login; does not create or remove accounts |
| **SCIM + SAML** | The full lifecycle pair — SCIM provisions accounts; SAML authenticates sessions |
| **Identity Drift** | The accumulation of stale, orphaned, or inconsistent identity data across systems when lifecycle events are managed manually |
| **Scheduled Import** | A periodic sync from the HR system to Okta; BambooHR does not support real-time webhook sync |

---

## Resources

- [Okta — BambooHR Integration Guide](https://help.okta.com/en-us/content/topics/provisioning/bamboohr/bamboohr-main.htm)
- [Okta — SCIM Provisioning Overview](https://developer.okta.com/docs/concepts/scim/)
- [Okta — Build a SCIM Provisioning Integration](https://developer.okta.com/docs/guides/scim-provisioning-integration-overview/)
- [Okta — Group Rules and Expression Language](https://help.okta.com/en-us/content/topics/users-groups-profiles/usgp-about-group-rules.htm)
- [Okta Expression Language Reference](https://developer.okta.com/docs/reference/okta-expression-language/)
- [SCIM 2.0 Specification (RFC 7644)](https://datatracker.ietf.org/doc/html/rfc7644)
- [BambooHR API Documentation](https://documentation.bamboohr.com/docs)

---

*Lab implementing a complete Joiner Mover Leaver identity lifecycle using BambooHR as the authoritative HR source, Okta as the identity broker with dynamic group rules, and a SterlingBank SCIM-provisioned application as the resource server — covering all three lifecycle phases end-to-end.*
