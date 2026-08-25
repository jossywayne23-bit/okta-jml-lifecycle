# Okta Identity Lifecycle Management — Joiner Mover Leaver (JML)


Hi, This lab is a hands-on implementation of a complete enterprise identity lifecycle using Okta as the Identity Provider, BambooHR as the authoritative HR source, and a SterlingBank application as the downstream resource server. This lab demonstrates automated user provisioning, attribute synchronisation, group-based access control, and deprovisioning — validated end-to-end using scim.dev as a live SCIM 2.0 test sink.

---
 
## Business Problem
 
Manual offboarding depends on a human filing a ticket and a second human actioning it. Every hour between those two events is an hour a terminated employee — or an attacker who obtained their credentials — retains standing access.
 
This is not a theoretical gap. Industry research consistently shows it is one of the most under-addressed risks in enterprise IAM:
 
| Finding | Source |
|---|---|
| Only 34% of organizations revoke system access on the day an employee leaves; for half of all organizations it takes three or more days | *Security Magazine* |
| 32% of organizations take seven or more days to fully deprovision a departing employee | OneLogin research |
| Over 70% of companies report instances of employees retaining inappropriate access after departure | SailPoint research |
 
**What this lab solves:** it replaces the manual HR-ticket-to-IT-action handoff with an automated chain — BambooHR (the single source of truth for employment status) drives Okta group membership, which drives SCIM provisioning into the resource server, with no manual step between an HR record change and an access change. The propagation window is reduced from "however long the ticket queue takes" to the BambooHR import interval — one hour in this lab, and further reducible in production if the HR platform supports webhook-driven sync instead of scheduled polling.
 
**Why this matters for the target sectors (financial services, healthcare):** stale privileged access after termination is a named control failure under SOC 2 CC6.2 (timely deprovisioning) and a specific audit point in HIPAA's Security Rule workforce security standard, which expects same-day revocation for healthcare portal access. An interviewer in either sector will ask "how fast can you prove access was removed" — this lab's answer is "the SCIM `PATCH /Users/{id}` timestamp in the scim.dev log," not "we hope the ticket was closed."
 
---

## System Architecture

<img src="jml-architecture.png" alt="Okta JML Automation Architecture" width="100%">
