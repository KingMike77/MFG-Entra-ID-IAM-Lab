# MFG (Mensah Financial Group) Entra ID IAM Lab

A hybrid identity lab for **Mensah Financial Group (MFG)**, a fictional regional bank. It solves the problems a bank's identity team faces: onboarding staff consistently, protecting admin and sensitive sign-ins without locking the tenant, bringing an on-premises Active Directory into the cloud, keeping privileged access temporary and auditable, and proving with logs what each access policy does.

It is built on Microsoft Entra ID plus an on-premises Active Directory in VMware, and organized around the **SC-300 (Microsoft Identity and Access Administrator)** objectives.

**Author:** Michael Mensah
**Timeline:** September 17th, 2026 - October 18th, 2026

> **About the evidence.** Several features used here (risk-based Conditional Access, Identity Protection, PIM, entitlement management, access reviews) need Entra ID P2 or Governance licensing. They were available through a Microsoft 365 E5 trial that ended on Oct 16, 2026, so those parts are documented with screenshots and log queries captured while the trial was live. The tenant now runs Entra ID Free and cannot run those features.

---

## The scenario

MFG has six departments, a small contractor workforce, and an on-premises Active Directory. The lab answers this brief:

- Give every employee an identity, a manager and the right group memberships from day one.
- Protect admin accounts and sensitive sign-ins without locking the tenant.
- Bring the on-premises directory and a domain-joined workstation into the cloud identity.
- Make privileged access temporary, justified and auditable.
- Let staff request application access through an approval workflow.
- Detect risky users and show, with logs, what each policy would have done.

## Lab environment

Two virtual machines run in **VMware Workstation** on a single host and stand in for MFG's on-premises network. They connect to a cloud tenant named **MensahFinancialGroup** (Microsoft Entra ID).

| Component | Role | OS | Resources | Notes |
|---|---|---|---|---|
| `MFG-DC01` | Domain controller and DNS | Windows Server 2022 | 2 vCPU, 2 GB RAM, 60 GB NVMe | Domain `mensahfinancial.local` (NetBIOS `MFG`), Windows Server 2016 domain functional level, NAT network |
| `MFG-WKS-0001` | Domain-joined workstation | Windows 11 | 2 vCPU, 4 GB RAM, 64 GB NVMe, vTPM | Joined to `mensahfinancial.local`, later hybrid Entra joined, NAT network |
| Tenant | Cloud identity | Microsoft Entra ID | n/a | `MensahFinancialGroup.onmicrosoft.com`. P2 features via a trial that ended Oct 16, 2026 |

![MFG Entra ID IAM architecture showing Active Directory, Entra Connect Sync, hybrid device registration, Microsoft Entra ID, and Log Analytics](images/MFG%20Entra%20ID%20IAM%20Lab%20Architecture%20%281%29.png)

**Tools used:** Microsoft Entra admin center, Microsoft 365 admin center, Microsoft Entra Connect Sync, Microsoft Graph PowerShell, Active Directory PowerShell, Azure Log Analytics (KQL).

## The phases

Each phase has its own folder with a README that explains what was done and why, with screenshots as evidence. They follow the order the lab was built.

| Phase | Focus | Outcome shown |
|---|---|---|
| [0. Environment setup](00-environment-setup/README.md) | The tenant, VMs and on-premises domain | Domain `mensahfinancial.local` running, workstation joined |
| [1. Identity foundation](01-identity-foundation/README.md) | Users, groups, units, licensing, emergency access | 41 users in the tenant (40 scripted), dynamic groups populated, 5 administrative units, 20 of 25 licences assigned |
| [2. Authentication and access control](02-authentication-and-access/README.md) | Methods, Conditional Access, SSPR, risk | 7 policies evaluated in report-only mode, a scenario evaluated with What If (expected outcome documented), a risky user raised to High |
| [3. Hybrid identity](03-hybrid-identity/README.md) | On-premises AD to Entra ID | 5 users and a group synced; workstation shows hybrid Entra joined |
| [4. Privileged access and governance](04-privileged-access-and-governance/README.md) | PIM, entitlement management | Eligible role activated with a time limit; access package configured |
| [5. Monitoring and workload identity](05-monitoring-and-workload-identity/README.md) | Logs, KQL, apps, attributes | Policy results queried in Log Analytics; app permission consented; attribute set defined |

## What was verified and what was not

"Completed" here means the phase was built. This table says how far each capability was actually checked.

| Label | Meaning |
|---|---|
| **Tested** | Configured and the result was observed (screenshot or log) |
| **Configured** | Set up, but no end-to-end test was run |
| **Limited** | Tested with a known limitation |
| **Not verified** | Attempted or planned, and not confirmed |

| Capability | Status | Detail |
|---|---|---|
| Scripted user creation | Tested | 40 users plus the admin account appear in the tenant, with manager set |
| Dynamic groups and administrative units | Tested | Rules processed successfully; AU membership populated by department |
| Group-based licensing | Tested | 20 of 25 licences assigned, three groups listed. The trial has since ended |
| Break-glass accounts | Configured | Two permanent Global Administrators. A recovery sign-in test was not run |
| Authentication methods, SSPR, banned passwords | Configured | Policies saved; no password reset was run through SSPR |
| Conditional Access (7 policies) | Limited | Report-only only. Results seen in logs and What If; no policy was enforced |
| Identity Protection | Tested | User flagged as compromised through the Graph API and shown as High risk |
| Entra Connect sync | Tested | 5 users and a group appear in Entra with on-premises sync enabled |
| Hybrid Entra join | Tested | `dsregcmd` and Entra Devices show the workstation as hybrid joined |
| Primary Refresh Token (PRT) | Not verified | Needs a sign-in to the workstation as a synced user |
| PIM | Limited | Eligible role activated and shown as active until expiry. The audit-log view was not captured |
| Entitlement management | Configured | Catalog and access package built. A request, approval and expiry were not demonstrated |
| Log Analytics and KQL | Tested | Query returns per-policy report-only results |
| App registration and consent | Tested | Application permission `Reports.Read.All` shown as not granted, then admin consent granted. The default delegated `User.Read` permission was consented in the same step |
| Custom security attributes | Limited | Attribute set and attributes defined. **Assigning values failed** with "insufficient privileges" and was not resolved |

## Licensing notes

Features in this lab sit at different licence levels. This follows Microsoft's published requirements as understood at the time of writing; check Microsoft's current documentation before relying on it.

| Capability | Licensing |
|---|---|
| Users, groups, administrative units, basic administration | Entra ID Free covers the foundations |
| Conditional Access | Generally Entra ID P1 or an eligible bundle |
| Risk-based Conditional Access, Identity Protection | Entra ID P2 |
| PIM, access reviews, entitlement management | Entra ID P2 or Entra ID Governance |

The current Free tenant does not provide the licensing entitlements needed to use the premium features documented in this lab. Their documentation here is evidence from when the trial was active.

## Security practices

- Break-glass accounts are cloud-only, permanent, and excluded from every Conditional Access policy so a policy error cannot lock the tenant. They are not used for routine administration. A production tenant would also need sign-in alerts and a periodic recovery test, which this lab did not set up. ([Phase 1](01-identity-foundation/README.md))
- Conditional Access policies were deployed in report-only mode and checked in logs. No policy was disabled to get around a block. ([Phase 2](02-authentication-and-access/README.md))
- Admin roles are eligible and time-bound where PIM was used. ([Phase 4](04-privileged-access-and-governance/README.md))
- Entra Connect uses a dedicated connector account rather than a domain administrator, and only one OU is synced. ([Phase 3](03-hybrid-identity/README.md))
- No passwords, tenant secrets or Temporary Access Pass codes are stored in this repository or shown in screenshots.

## SC-300 coverage

| Exam domain | Covered in |
|---|---|
| Implement identities in Microsoft Entra ID | Phases 1 and 3, plus custom security attributes in Phase 5 |
| Implement authentication and access management | Phase 2 |
| Plan and implement workload identities | Phase 5 |
| Plan and implement identity governance | Phase 4, plus Terms of use in Phase 2 and logging in Phase 5 |

---

*This is a fictional organization built for learning. All names are invented.*
