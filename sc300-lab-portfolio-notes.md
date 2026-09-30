# Enterprise Identity & Access Lab — Microsoft Entra ID (SC-300)

**Author:** Steve (Payne)
**Status:** In progress
**Purpose:** A hands-on Microsoft identity and access management lab built to (1) prepare for the Microsoft SC-300: Identity and Access Administrator exam and (2) serve as a demonstrable cloud identity portfolio project.

> Every resource in this lab was built by hand in a personal tenant. This document records the architecture, the design decisions and their rationale, the build steps, and the problems solved.

---

## 1. Objective & Scope

Design and build a realistic enterprise-style identity environment in Microsoft Entra ID, covering the four SC-300 domains: user identities, authentication & access management, workload identities, and identity governance. A local Windows Server Active Directory forest is added later to demonstrate hybrid identity via Entra Cloud Sync.

Scope is deliberately bounded to SC-300 objectives. Anything outside them (e.g. Microsoft Sentinel) is marked optional and treated as an add-on.

---

## 2. Architecture (summary)

```
On-prem VMware lab (added later)          Microsoft Entra ID tenant (cloud)
  - DC01: Windows Server AD DS + DNS  ──►    - Users, groups, admin roles, devices
  - CLIENT01: Win 11 domain-joined           - External identities (B2B)
  - Entra Cloud Sync agent                    - Authentication: MFA, SSPR, Conditional Access
                                              - Authorization: Azure RBAC, PIM, Governance
                                              - Enterprise apps / app registrations
                                                        │
                                                        ▼
                                        Monitoring: sign-in/audit logs →
                                        Log Analytics → KQL (+ optional Sentinel)
```

*(Replace this block with the exported architecture diagram image once added.)*

---

## 3. Environment

| Layer | Detail |
|---|---|
| Cloud tenant | Microsoft Entra ID (initial domain `*.onmicrosoft.com`) |
| Azure | Free account (subscription for RBAC / Log Analytics / KQL) |
| Licensing plan | Entra ID Free tier now; Entra ID P2 free trial to be activated for premium features |
| On-prem (later) | VMware — Windows Server AD DS, Windows 11 client |

---

## 4. Design Decisions & Rationale

This section is the point of the document — it shows reasoning, not just clicks.

- **Cost model: free-first, trial-timed.** Built on the Entra Free tier + Azure free account ($0). Premium features (Conditional Access, PIM, Access Reviews, Entitlement Management, Identity Protection) require Entra ID P1/P2, so they are scheduled into a focused block during a one-time 30-day P2 trial rather than spread out, to avoid wasting the non-renewable trial window.
- **Build order: cloud premium features first, on-prem AD last.** Hybrid identity is the lowest-weighted SC-300 area and is free/un-clocked, so it is sequenced after the premium, higher-weighted governance work.
- **User attributes set at creation (department, job title, usage location).** These are prerequisites for dynamic group rules and licensing. Usage location in particular is required before any license can be assigned.
- **Group strategy: dynamic vs assigned, chosen per purpose.**
  - `SG-Sales-Dynamic` — *dynamic* membership (`user.department -eq "Sales"`), so access follows role changes automatically (joiner/mover/leaver handled by rule).
  - `SG-IT-Admins` — *assigned* membership, because privileged/admin group membership should be deliberate and curated, not rule-driven.
- **Naming conventions:** users `first.last`; security groups `SG-<function>-<type>`. Consistent naming is treated as an operational requirement, not cosmetic.
- **Security hygiene:** no credentials or real recovery contacts committed to the repo; test identities only.

---

## 5. Build Log

### 2026-09-21 — Tenant & subscription provisioning
- Created a new Microsoft account and an Azure free account, provisioning a new Entra tenant and an Azure subscription in one step.
- Confirmed the tenant/subscription split and the corresponding roles: **Global Administrator** (Entra tenant) vs **Owner** (Azure subscription).
- Verified the portal shows an active subscription and a loadable Entra tenant.

### 2026-09-21 — User provisioning & first-logon flow
- Created three member users with department, job title, and usage location:
  - Jordan Reed — Account Executive, Sales
  - Maria Chen — Systems Administrator, IT
  - Sam Patel — Financial Analyst, Finance
- Performed an **admin password reset** on a test user and confirmed the **forced password change on first sign-in**, verifying end-to-end provisioning.
- Noted the distinction between an admin-issued temporary password and a user-set secret (baseline for later SSPR / MFA / passwordless work).

### 2026-09-29 — Security groups
- Created `SG-IT-Admins` (assigned) with Maria Chen as a member.
- Created `SG-Sales-Dynamic` (dynamic user, rule `user.department -eq "Sales"`).
- Observed the dynamic group does not populate on the Free tier — dynamic membership requires Entra ID P1+. Documented as a known licensing gate to re-verify after the P2 trial is active.

*(Continue appending dated entries per phase.)*

---

## 6. Skills Demonstrated (mapped to SC-300)

Honest, current-state list — grows as the lab progresses.

- **Domain 1 — User identities:** tenant setup; member user creation with attributes; usage location for licensing; assigned vs dynamic security groups; dynamic membership rule authoring.
- **Domain 2 — Authentication & access:** admin password reset and forced-change flow (foundation for SSPR/MFA — pending).
- **Governance/licensing awareness:** identified P1/P2 feature gating and planned around trial constraints.

*Pending (not yet built — do not claim until done): Conditional Access, MFA, SSPR, PIM, Access Reviews, Entitlement Management, app registrations/workload identities, Azure RBAC, hybrid identity via Cloud Sync, KQL monitoring.*

---

## 7. Troubleshooting Log

| Date | Symptom | Cause | Resolution |
|---|---|---|---|
| 2026-09-21 | "Account or password is incorrect" at sign-in for test user | Auto-generated password shown only once was not captured | Performed admin password reset; switched to admin-set known password for lab test users |

*(Append as issues arise — this table is high-signal for interviews.)*

---

## 8. Resume Bullets (draft — accurate to current progress)

Keep these truthful and update as the lab grows. Current state supports:

- Built a Microsoft Entra ID tenant from scratch and provisioned member identities with attribute-based structure (department, job title, usage location) to support downstream dynamic access and licensing.
- Implemented security group strategy using both dynamic (attribute-rule) and assigned membership, applying least-privilege reasoning to privileged group design.
- Documented cloud identity architecture, licensing constraints, and cost controls for a self-directed SC-300 lab.

*Stronger bullets (Conditional Access, PIM, governance, hybrid, KQL) unlock as those phases complete — add them then, not before.*

---

## 9. Next Steps

- Assign an Entra admin role to a group (least-privilege) → sets up PIM.
- Activate Entra ID P2 trial; verify dynamic group populates.
- Configure MFA and SSPR.
- Build Conditional Access policies (report-only first).
- Add on-prem AD DS + Entra Cloud Sync (hybrid identity).
- Stand up Log Analytics + KQL for sign-in monitoring (schedule before Azure credit expiry).
