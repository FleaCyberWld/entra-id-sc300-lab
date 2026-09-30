# Enterprise Identity & Access Lab — Microsoft Entra ID (SC-300)

**Author:** Steve (Payne)
**Status:** In progress — Free-tier foundation complete
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
| Licensing | Entra ID Free tier; Entra ID P2 free trial planned for premium features |
| Tenant security baseline | Security Defaults enabled (enforces MFA registration) |
| On-prem (later) | VMware — Windows Server AD DS, Windows 11 client |

---

## 4. Design Decisions & Rationale

- **Cost model: free-first, trial-timed.** Built on the Entra Free tier + Azure free account ($0). Premium features (Conditional Access, PIM, Access Reviews, Entitlement Management, Identity Protection, dynamic groups, role-assignable groups) require Entra ID P1/P2, so they are batched into a single 30-day P2 trial rather than spread out.
- **Build order: cloud premium features first, on-prem AD last.** Hybrid identity is the lowest-weighted SC-300 area and is free/un-clocked, so it is sequenced after the higher-weighted governance work.
- **User attributes set at creation (department, job title, usage location).** Prerequisites for dynamic group rules and licensing. Usage location is required before any license can be assigned.
- **Naming conventions:** users `first.last`; security groups `SG-<function>-<type>`. Consistent naming is treated as an operational requirement, not cosmetic.
- **Dynamic vs assigned groups, chosen per purpose.**
  - `SG-Sales-Dynamic` — intended *dynamic* membership (`user.department -eq "Sales"`) so access follows role changes automatically (joiner/mover/leaver handled by rule, no ticket).
  - `SG-IT-Admins` — *assigned* membership, because privileged/admin group membership should be deliberate and curated, not rule-driven. A rule-based admin group means anyone whose department attribute changes silently gains admin rights.
- **Least privilege for admin roles.** Delegated user management with the scoped built-in **User Administrator** role instead of Global Administrator. Target end state: role assigned to a role-assignable group, then converted to a PIM eligible (just-in-time) assignment.
- **Emergency access account.** A cloud-only Global Administrator on the `.onmicrosoft.com` domain, independent of the primary admin (which is a personal Microsoft account managed outside Entra). Will be excluded from all Conditional Access policies to prevent tenant lockout.
- **Security hygiene:** no credentials, emergency-account identifiers, or real recovery contacts in this repo; test identities only.

---

## 5. Build Log

### 2026-09-21 — Tenant & subscription provisioning
- Created a new Microsoft account and an Azure free account, provisioning an Entra tenant and an Azure subscription in one step.
- Confirmed the tenant/subscription split and the corresponding roles: **Global Administrator** (Entra tenant) vs **Owner** (Azure subscription).

### 2026-09-21 — User provisioning & first-logon flow
- Created three member users with department, job title, and usage location:
  - Jordan Reed — Account Executive, Sales
  - Maria Chen — Systems Administrator, IT
  - Sam Patel — Financial Analyst, Finance
- Performed an admin password reset on a test user and confirmed the forced password change on first sign-in.

### 2026-09-29 — Security groups
- Created `SG-IT-Admins` (Security, Assigned) with Maria Chen as a member.
- Attempted to create `SG-Sales-Dynamic` as Dynamic User; the rule editor returned HTTP 403 (see Troubleshooting).
- Created `SG-Sales-Dynamic` as an **Assigned, empty placeholder** on the Free tier, to be converted to Dynamic User once the P2 trial is active. Left empty deliberately: converting Assigned → Dynamic clears existing members and repopulates from the rule.

### 2026-09-29 — Admin role assignment (least privilege)
- Confirmed role-assignable groups are unavailable on the Free tier: the "Microsoft Entra roles can be assigned to the group" option is absent from the New group form without P1/P2.
- Assigned the built-in **User Administrator** role **directly** to Maria Chen as an interim state.
- Verified the assignment from both directions: the role's Assignments page and the user's Assigned roles.
- Planned migration after P2: direct assignment → role-assignable `SG-IT-Admins` → PIM eligible (just-in-time).

### 2026-09-29 — Role-scope testing
- **Test 1:** Maria (User Administrator) reset Jordan Reed's password (non-admin user). **Result: succeeded.** Role permission confirmed.
- **Test 2:** Maria attempted to reset the primary Global Admin's password. **Result:** "is a Microsoft account that is managed by the user." This is an account-type block, not a role-scope denial: the primary admin is a personal Microsoft account whose credentials are managed outside Entra, so no one in the tenant can reset it. Role-scope test moved to a cloud-only admin target.
- **Test 3:** Maria attempted to reset the cloud-only emergency access account (Global Administrator). **Result:** "The password can not be reset. This may be due to an incorrect level of administrative privilege..." Confirms User Administrator cannot reset passwords of higher-privileged admins.

### 2026-09-29 — Emergency access account
- Created a cloud-only emergency access account with Global Administrator. Credentials stored offline; identifiers excluded from this repo.
- Global Administrator assignment count verified at 2.
- MFA registered via Microsoft Authenticator (registration enforced by Security Defaults).
- **Known gap:** the Authenticator shares a phone with the primary admin's recovery path, so losing one device affects both admin paths. Planned fix: add an independent method (FIDO2 security key or a separate device).

*(Continue appending dated entries per phase.)*

---

## 6. Skills Demonstrated (mapped to SC-300)

Current state only — grows as the lab progresses.

- **Domain 1 — User identities:** tenant setup; member users with attributes; usage location for licensing; assigned security groups; dynamic rule design (pending license).
- **Domain 2 — Authentication & access:** admin password reset and forced-change flow; MFA registration under Security Defaults; built-in admin role assignment with least privilege; role-scope verification; emergency access account.
- **Licensing awareness:** identified and documented P1/P2 feature gates (dynamic groups, role-assignable groups, custom roles) and planned around them.

*Pending (not yet built, do not claim until done): Conditional Access, SSPR, PIM, Access Reviews, Entitlement Management, app registrations/workload identities, Azure RBAC, hybrid identity via Cloud Sync, KQL monitoring.*

---

## 7. Troubleshooting Log

| Date | Symptom | Cause | Resolution |
|---|---|---|---|
| 2026-09-21 | "Account or password is incorrect" at sign-in for a test user | Auto-generated password is displayed only once and was not captured | Admin password reset; admin-set known passwords for lab test users |
| 2026-09-29 | HTTP 403 on `DynamicGroupV2Blade` when selecting Add dynamic query, despite Global Administrator rights | Dynamic membership requires Entra ID P1+; the Free tier blocks the rule editor | Created group as Assigned placeholder; conversion deferred to P2 phase |
| 2026-09-29 | Role-assignable group not selectable when assigning User Administrator (no Groups tab in Add assignments) | Role-assignable groups require P1+; option absent at group creation on Free tier | Assigned role directly to user as interim state |
| 2026-09-29 | Two different password-reset denials | (1) Microsoft-account admin blocked by **account type**; (2) cloud Global Admin blocked by **role scope** | Distinguished root causes by error text; documented both |

---

## 8. Resume Bullets (draft — accurate to current progress)

- Built a Microsoft Entra ID tenant from scratch and provisioned member identities with attribute-based structure (department, job title, usage location) to support dynamic access and licensing.
- Applied least-privilege administration by delegating user management through the scoped User Administrator role rather than Global Administrator, and verified role-scope enforcement through positive and negative password-reset tests.
- Implemented a cloud-only emergency access (break-glass) Global Administrator account to eliminate a single point of failure in tenant administration.
- Diagnosed Entra ID licensing gates (HTTP 403 on dynamic group rules, unavailable role-assignable groups) and documented root cause and remediation.

*Stronger bullets (Conditional Access, PIM, governance, hybrid, KQL) unlock as those phases complete. Add them then, not before.*

---

## 9. Next Steps

- Decide timing of Entra ID P2 trial (one-time, 30 days).
- After P2: convert `SG-Sales-Dynamic` to Dynamic User and verify it auto-populates; recreate `SG-IT-Admins` as role-assignable and move User Administrator to it; convert to PIM eligible.
- Configure SSPR and MFA methods; build Conditional Access policies (report-only first), excluding the emergency access account.
- Add an independent MFA method to the emergency access account.
- Add on-prem AD DS + Entra Cloud Sync (hybrid identity).
- Stand up Log Analytics + KQL before the Azure credit expires (~Oct 21).
