# Enterprise Identity & Access Lab — Microsoft Entra ID (SC-300)

A hands-on Microsoft **Entra ID** identity and access management lab, built from scratch in a personal tenant. It doubles as SC-300 (*Microsoft Identity and Access Administrator*) exam preparation and a demonstrable cloud identity portfolio project.

> **All resources were built and configured by hand.** This repo documents the architecture, the design decisions and their reasoning, a dated build log, and the problems solved along the way — not just screenshots.

---

## What this lab covers

Scoped deliberately to the four SC-300 domains:

- **User identities** — tenant setup, member users with attribute structure, assigned vs dynamic security groups
- **Authentication & access** — MFA, SSPR, Conditional Access *(in progress)*
- **Workload identities** — app registrations, service principals *(planned)*
- **Identity governance** — PIM, Access Reviews, Entitlement Management *(planned)*

Hybrid identity (on-prem Windows Server AD DS synced via Entra Cloud Sync) is added in a later phase.

---

## Architecture

```
On-prem VMware lab (later)              Microsoft Entra ID tenant (cloud)
  DC01: AD DS + DNS          ──►          Users · Groups · Admin roles · Devices
  CLIENT01: Win 11 joined                 External identities (B2B)
  Entra Cloud Sync agent                  Auth: MFA · SSPR · Conditional Access
                                          Authz: RBAC · PIM · Governance · Apps
                                                       │
                                                       ▼
                                      Sign-in/audit logs → Log Analytics → KQL
                                                  (+ optional Sentinel)
```

*(Diagram image to be added: `docs/architecture.png`)*

---

## Design decisions (why, not just what)

- **Free-first, trial-timed licensing.** Built on Entra Free + Azure free account; premium features (Conditional Access, PIM, governance) are batched into a single 30-day Entra ID P2 trial to avoid wasting a non-renewable window.
- **Cloud premium features first, on-prem AD last** — hybrid is the lowest-weighted SC-300 area and is free/un-clocked, so it's sequenced after the higher-value work.
- **Dynamic vs assigned groups, chosen per purpose** — `SG-Sales-Dynamic` uses an attribute rule so access follows role changes; `SG-IT-Admins` is assigned because privileged membership should be deliberate.
- **Security hygiene** — test identities only; no credentials or real recovery contacts in this repo.

Full reasoning and the running record are in **[the build log](./sc300-lab-portfolio-notes.md)**.

---

## Tech

Microsoft Entra ID · Azure (RBAC, Log Analytics, KQL) · Windows Server AD DS · Entra Cloud Sync · PowerShell · Conditional Access · PIM

---

## Status

**In progress.** See the [build log](./sc300-lab-portfolio-notes.md) for dated entries and current phase.

---

*This is an isolated training lab. All users and data are fictitious.*
