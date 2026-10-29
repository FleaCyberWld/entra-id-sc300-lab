# Enterprise Identity & Access Lab — Microsoft Entra ID (SC-300)

**Author:** Steve (Payne)
**Status:** In progress — Free-tier foundation complete; Microsoft Graph PowerShell automation in progress
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
| Licensing | Entra ID Free tier; Entra ID P2 free trial planned after Nov 21, 2026 |
| Tenant security baseline | Security Defaults enabled (enforces MFA registration) |
| Azure subscription | Free credit expired Oct 2026; subscription disabled, no charges. Upgrade deferred to Azure RBAC / Log Analytics phase |
| Automation | PowerShell 7.6.6; Microsoft Graph PowerShell SDK 2.41.2 (Authentication, Users, Groups, Identity.DirectoryManagement modules only) |
| Repo layout | `scripts/01-connect`, `scripts/02-bulk-users`, `docs/screenshots`, `kql` |
| On-prem (later) | VMware — Windows Server AD DS, Windows 11 client |

---

## 4. Design Decisions & Rationale

- **Cost model: free-first, trial-timed.** Built on the Entra Free tier + Azure free account ($0). Premium features (Conditional Access, PIM, Access Reviews, Entitlement Management, Identity Protection, dynamic groups, role-assignable groups) require Entra ID P1/P2, so they are batched into a single 30-day P2 trial, scheduled around travel so the trial window isn't wasted.
- **Build order: cloud premium features first, on-prem AD last.** Hybrid identity is the lowest-weighted SC-300 area and is free/un-clocked.
- **User attributes set at creation (department, job title, usage location).** Prerequisites for dynamic group rules and licensing. Usage location is required before any license can be assigned.
- **Naming conventions:** users `first.last`; security groups `SG-<function>-<type>`.
- **Dynamic vs assigned groups, chosen per purpose.**
  - `SG-Sales-Dynamic` — intended *dynamic* membership (`user.department -eq "Sales"`) so access follows role changes automatically.
  - `SG-IT-Admins` — *assigned* membership, because privileged group membership should be deliberate. A rule-based admin group means anyone whose department attribute changes silently gains admin rights.
- **Least privilege for admin roles.** Delegated user management with the scoped built-in **User Administrator** role instead of Global Administrator. Target end state: role-assignable group, then PIM eligible (just-in-time).
- **Emergency access account.** Cloud-only Global Administrator on the `.onmicrosoft.com` domain, independent of the primary admin (a personal Microsoft account managed outside Entra). Credential kept in a password manager with autofill disabled, MFA on a dedicated second device, never used for routine work, and to be excluded from all Conditional Access policies.
- **Least privilege in automation.** Graph PowerShell connections request a minimal, task-specific set of delegated scopes, pin the tenant, and compare requested vs. granted scopes, because delegated tokens carry every scope previously consented for that app and user.
- **Install only what's used.** Four Graph SDK modules instead of the full 40+ module bundle.
- **Security hygiene:** no credentials, emergency-account identifiers, or real recovery contacts in this repo. `.gitignore` excludes `output/`, `*.secret.csv`, `*passwords*`, and `*.log`.

---

## 5. Build Log

### 2026-09-21 — Tenant & subscription provisioning
- Created a new Microsoft account and an Azure free account, provisioning an Entra tenant and an Azure subscription in one step.
- Confirmed the tenant/subscription split: **Global Administrator** (Entra tenant) vs **Owner** (Azure subscription).

### 2026-09-21 — User provisioning & first-logon flow
- Created three member users with department and job title:
  - Jordan Reed — Account Executive, Sales
  - Maria Chen — Systems Administrator, IT
  - Sam Patel — Financial Analyst, Finance
- Performed an admin password reset on a test user and confirmed the forced password change on first sign-in.

### 2026-09-29 — Security groups
- Created `SG-IT-Admins` (Security, Assigned) with Maria Chen as a member.
- Attempted `SG-Sales-Dynamic` as Dynamic User; the rule editor returned HTTP 403 (see Troubleshooting).
- Created `SG-Sales-Dynamic` as an **Assigned, empty placeholder**, to be converted after P2. Left empty deliberately: converting Assigned → Dynamic clears existing members.

### 2026-09-29 — Admin role assignment (least privilege)
- Confirmed role-assignable groups are unavailable on the Free tier (the option is absent from the New group form without P1/P2).
- Assigned the built-in **User Administrator** role **directly** to Maria Chen as an interim state; verified from both the role and the user object.
- Planned migration after P2: direct assignment → role-assignable `SG-IT-Admins` → PIM eligible.

### 2026-09-29 — Role-scope testing
- **Test 1:** Maria (User Administrator) reset Jordan Reed's password (non-admin). **Succeeded.**
- **Test 2:** Maria attempted to reset the primary Global Admin. **Blocked by account type:** "is a Microsoft account that is managed by the user." No one in the tenant can reset it.
- **Test 3:** Maria attempted to reset the cloud-only emergency account (Global Administrator). **Blocked by role scope:** "...incorrect level of administrative privilege..."

### 2026-09-29 — Emergency access account
- Created a cloud-only emergency access account with Global Administrator. Global Administrator count verified at 2.
- MFA registered via Microsoft Authenticator (enforced by Security Defaults).

### 2026-10-25 — Pre-trip verification
- **Azure:** free credit expired; subscription disabled, no charges incurred. Entra tenant unaffected. Upgrade to pay-as-you-go deferred; a budget alert is required on upgrade.
- **Tenant audit:** domain, Free license, groups, role counts (Global Administrator 2, User Administrator 1), and Security Defaults all as expected.
- **Usage location** found blank on 4 of 5 users (department and job title were present), so it was skipped during manual creation. No impact until licensing. Remediation planned via Graph PowerShell; the bulk-create script will set it by default.
- **Emergency account credential test failed:** the password had not been stored. Reset by the primary Global Admin; sign-in and MFA verified. Lesson: an emergency account that has never been tested is a false sense of security.

### 2026-10-27 — Emergency account hardening & repo structure
- Emergency credential moved from plain notes into a password manager.
- Independent MFA method registered on a dedicated second device kept at home, removing the shared-phone dependency with the primary admin's recovery path.
- Created repo folders (`scripts/01-connect`, `scripts/02-bulk-users`, `docs/screenshots`, `kql`) and added secret/output exclusions to `.gitignore`.

### 2026-10-27 — Microsoft Graph PowerShell
- Installed PowerShell 7.6.6 and four Microsoft Graph SDK modules (v2.41.2), scoped to the current user.
- Built `Connect-LabGraph.ps1`: task-based minimal delegated scopes (`ReadOnly`, `ManageUsers`, `BulkUsers`), pinned tenant, fresh-session disconnect, and a requested-vs-granted scope comparison.
- Connected in `ReadOnly` mode. Granted scopes matched the request exactly (plus standard sign-in scopes); tenant ID verified against the portal.

### 2026-10-28 — Identity verification incident
- The first successful Graph session authenticated as the **emergency access account** instead of the primary admin. The SDK context did not display the account name, so it was detected through tenant-wide sign-in logs.
- Root cause: the password manager offered the emergency credential on the Microsoft sign-in page.
- Remediation: disconnected, re-authenticated explicitly as the primary admin, and confirmed identity by matching the sign-in log **User ID** to the primary admin's **Object ID** (names differ for Microsoft-account users, IDs don't).
- Follow-ups: disable autofill on the emergency credential; remove the emergency account's consent grant on Microsoft Graph Command Line Tools; make the connection script warn when the account can't be identified.

*(Continue appending dated entries per phase.)*

---

## 6. Skills Demonstrated (mapped to SC-300)

Current state only — grows as the lab progresses.

- **Domain 1 — User identities:** tenant setup; member users with attributes; assigned security groups; dynamic rule design (pending license); attribute audit.
- **Domain 2 — Authentication & access:** admin password reset and forced-change flow; MFA under Security Defaults; least-privilege role assignment with positive and negative role-scope tests; emergency access account with independent MFA and periodic credential testing.
- **Domain 3 — Workload identities (early):** delegated Graph permissions and consent; least-privilege scope design for automation.
- **Monitoring (early):** used Entra sign-in logs to detect and attribute an unexpected privileged sign-in, matching on object IDs.
- **Licensing awareness:** documented P1/P2 feature gates (dynamic groups, role-assignable groups, custom roles).

*Pending (not yet built, do not claim until done): Conditional Access, SSPR, PIM, Access Reviews, Entitlement Management, app registrations, Azure RBAC, hybrid identity via Cloud Sync, KQL monitoring.*

---

## 7. Troubleshooting Log

| Date | Symptom | Cause | Resolution |
|---|---|---|---|
| 2026-09-21 | "Account or password is incorrect" for a test user | Auto-generated password is shown only once and wasn't captured | Admin password reset; admin-set lab passwords |
| 2026-09-29 | HTTP 403 on `DynamicGroupV2Blade` when selecting Add dynamic query, despite Global Administrator rights | Dynamic membership requires P1+ | Assigned placeholder; conversion deferred to P2 |
| 2026-09-29 | Group not selectable when assigning User Administrator | Role-assignable groups require P1+ | Direct user assignment as interim state |
| 2026-09-29 | Two different password-reset denials | (1) Microsoft-account admin blocked by **account type**; (2) cloud Global Admin blocked by **role scope** | Distinguished root causes by error text |
| 2026-10-25 | Usage location blank on 4 of 5 users | Field skipped during manual creation (other Properties fields saved) | Fix via Graph PowerShell; default it in bulk script |
| 2026-10-25 | Could not sign in to emergency account | Credential never stored after creation | Reset by Global Admin; stored in password manager; re-tested |
| 2026-10-27 | `Connect-MgGraph`: "User canceled authentication" | Graph SDK uses Windows Web Account Manager (WAM), a native dialog the Bitwarden browser extension can't autofill | Disabled WAM so sign-in uses the browser |
| 2026-10-27 | `Set-MgGraphOption -EnableLoginByWAM` not found | Parameter name differs in installed version | Inspected the cmdlet's parameters; correct name was `DisableLoginByWAM` (Nullable Boolean) |
| 2026-10-27 | Passkey sign-in failed via QR code | Phone routed the passkey request to Bitwarden, which doesn't hold the Microsoft passkey | Used password + MFA instead; passkey provider fix pending |
| 2026-10-28 | Graph session ran as emergency account; SDK showed no account name | Password manager offered the emergency credential | Re-authenticated; verified identity via sign-in log User ID = Object ID |
| 2026-10-28 | Per-user sign-in log view showed no entries for the emergency account | Entry was on the non-interactive tab or delayed; per-user view is filtered | Used tenant-wide sign-in logs. Lesson: an empty log view isn't proof nothing happened |

---

## 8. Resume Bullets (draft — accurate to current progress)

- Built a Microsoft Entra ID tenant from scratch and provisioned member identities with attribute-based structure to support dynamic access and licensing.
- Applied least-privilege administration by delegating user management through the scoped User Administrator role rather than Global Administrator, and verified role-scope enforcement with positive and negative tests.
- Implemented and hardened a cloud-only emergency access Global Administrator account (independent MFA device, password-manager storage, periodic credential testing).
- Wrote a Microsoft Graph PowerShell connection script that requests task-specific minimal delegated scopes and validates granted vs. requested permissions.
- Investigated an unexpected privileged sign-in using Entra sign-in logs, attributing it by object ID and documenting root cause and remediation.
- Diagnosed Entra ID licensing gates (HTTP 403 on dynamic group rules, unavailable role-assignable groups) and documented root cause and remediation.

*Stronger bullets (Conditional Access, PIM, governance, hybrid, KQL) unlock as those phases complete.*

---

## 9. Next Steps

- Update `Connect-LabGraph.ps1` to warn when the signed-in account can't be identified; commit it to `scripts/01-connect`.
- Verify the `ReadOnly` scope blocks writes (`Update-MgUser` test).
- Remove the emergency account's consent grant on Microsoft Graph Command Line Tools.
- Fix the four blank usage locations with `-Task ManageUsers`.
- Build the CSV bulk-create script (`scripts/02-bulk-users`) with README.
- After P2: convert `SG-Sales-Dynamic`; recreate `SG-IT-Admins` as role-assignable; PIM; SSPR; Conditional Access excluding the emergency account.
- Add on-prem AD DS + Entra Cloud Sync. Upgrade Azure to pay-as-you-go (with budget alert) for RBAC and Log Analytics + KQL.
