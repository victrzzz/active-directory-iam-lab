# Active Directory + Okta IAM Lab

I built a self-hosted identity and access management lab: a Windows Server domain controller, a domain-joined Windows 11 admin workstation, role-based access to file shares, Group Policy, and an Okta org with MFA. Then I tested that access follows group membership and ran a simulated access review on 40 accounts.

Everything runs on one mini PC under Proxmox. This README documents what I built, what I tested, what broke, and how I fixed it.

> **Status:** In progress. The directory, shares, GPO and Okta work are done and tested. The access review is complete; remediation in AD is next. See [Where it stands](#where-it-stands).

---

## What I built

- **A Windows Server 2022 domain controller** (Server Core, no GUI) for `lab.local`, with DNS, running as a VM on Proxmox.
- **A Windows 11 Enterprise client**, domain-joined, managed through RSAT (Active Directory Users and Computers, Group Policy Management) and reachable over OpenSSH.
- **A directory for a fictional company, "Harborview Logistics":** 4 OUs (Finance, HR, IT, Security Groups), 20 role-based security groups, and 40 users with realistic job titles and memberships. I created the first user and groups by hand in ADUC, then wrote an idempotent PowerShell script ([`scripts/Build-HarborviewAD.ps1`](scripts/Build-HarborviewAD.ps1)) for the rest.
- **Department file shares** (`\\DC01\Finance`, `\\DC01\HR`, `\\DC01\IT`) where NTFS permissions are granted only to that department's group. I built Finance by hand in the GUI, then scripted HR and IT.
- **A GPO** linked to the Finance OU that maps the Finance share as `F:` at logon.
- **An Okta org** (Integrator Free Plan) with MFA via Okta Verify, three groups mirroring AD, six users, and two apps assigned to the Finance group only.
- **A simulated access review dataset:** 40 accounts with job titles, managers, employment type, last login, systems, group memberships and business justifications, seeded with the kinds of problems a real review exists to catch.

---

## Architecture

```
                     Proxmox Host (HP EliteDesk 800 G2 Mini)
                     Intel i5-6500T · 16GB RAM · 256GB NVMe VM storage
  ┌──────────────────────────────────────────────────────────────┐
  │                                                                │
  │   ┌─────────────────────────┐      ┌────────────────────────┐ │
  │   │  DC01  (VM 101)          │      │  Client  (VM 102)      │ │
  │   │  Windows Server 2022     │◄────►│  Windows 11 Enterprise │ │
  │   │  Server Core (no GUI)    │ domain│  Domain-joined         │ │
  │   │  Domain Controller       │ join  │  RSAT (ADUC/GPMC/DNS)  │ │
  │   │  DNS for lab.local       │      │  OpenSSH Server        │ │
  │   │  192.168.1.160           │      │  192.168.1.161         │ │
  │   └─────────────────────────┘      └────────────────────────┘ │
  │                                                                │
  └──────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │   Okta (cloud)    │  groups mirrored by hand
                    │ Integrator Free   │  (group-based app access)
                    └───────────────────┘
```

**Domain:** `lab.local`

**Why Server Core:** I ran the DC without a desktop GUI for a smaller footprint and attack surface. I administer it remotely from the Windows 11 client with RSAT, the way admins manage DCs from a workstation instead of signing in to the server.

---

## Access model

Access is assigned to groups, never to individuals.

- **OUs** organize users by department and scope Group Policy. They don't grant access.
- **Security groups** (for example `Finance-Users`, `Finance-Managers`, `Payroll-Processors`, `HRIS-Admins`, `IT-Helpdesk`, `Server-Admins`) carry the permissions.
- **File shares** are open at the share level, and NTFS does the gatekeeping: inheritance removed, and only Administrators, SYSTEM and the department's `-Users` group (Modify) can get in.
- **Group Policy** decides who receives a setting. The `Finance - Map Share` GPO is linked to the Finance OU, so Finance users get the `F:` drive, but only members of `Finance-Users` can open it.
- **Okta** has no OUs, so access there is driven by groups. I mirrored `Finance-Users`, `HR-Users` and `IT-Users` by hand and assigned two apps to the Finance group.

---

## Results

I tested that access follows group membership, on both the Windows side and the Okta side.

| Test | Expected | Result |
|---|---|---|
| Finance user (`premington`) opens `\\DC01\Finance` | Allowed | ✅ Allowed |
| Same user opens `\\DC01\HR` | Denied | ✅ Access is denied |
| Removed from `Finance-Users`, fresh login opens `\\DC01\Finance` | Denied | ✅ Access is denied |
| Added back to `Finance-Users`, fresh login | Allowed | ✅ Allowed |
| Finance user signs in at the console | `F:` drive appears and opens | ✅ Appears and opens |
| HR user signs in at the console | No `F:` drive | ✅ No `F:` |
| Finance user signs in to Okta | Assigned apps appear | ✅ Two apps appear |
| HR user signs in to Okta | No apps | ✅ Empty dashboard |

Evidence is in [`screenshots/`](screenshots/): for example `premington_denied_access.png`, `premington_oktaapps.png` and `bcastellano_oktaapps.png`.

---

## Access review

I reviewed all 40 accounts as if I were the access reviewer: for each one I checked job title, manager, employment type, last login, systems and group memberships against the business justification, then recorded a decision, what to remove, who owns the decision, and why. The workbook is in [`docs/Harborview_Access_Review.xlsx`](docs/Harborview_Access_Review.xlsx).

| Decision | Accounts |
|---|---|
| Keep | 32 |
| Modify access | 3 |
| Remove access | 1 |
| Disable account | 3 |
| Escalate | 1 |

**What I found**

| Account | Issue | Decision |
|---|---|---|
| Wesley Crane | Terminated 2026-08-14, account still enabled (offboarding gap) | Disable account |
| Derek Vance | Contractor ended 2026-06-30, logged in 2026-09-12 after the end date | Disable account, investigate activity |
| Maya Ortega | Contractor ended 2026-09-30, logged in 2026-10-21 with admin access (IT-Admins, Network-Ops) | Disable account, review admin activity since contract end |
| Ian Pruitt | Still in Domain Admins from a 2025 project | Remove Domain Admins and AD Admin Console, keep IT-Helpdesk |
| Sam Delgado | Kept manager and admin access from covering a 2024 leave | Modify: remove Finance-Managers, Expenses-Approvers, ERP-Finance (Admin), Payroll |
| Jordan Ellis | Moved from Finance to HR, still holds Finance access | Modify: remove Finance-Users, Finance-Reports, Expenses-Approvers, ERP-Finance, Expense Portal |
| Ruth Calloway | HR generalist with leftover payroll access from a one-time correction | Modify: remove Payroll-Processors and Payroll |
| Heidi Lawson | No login since 2026-02-03 | Escalate to manager; disable if unused |
| Ryan Kowalski | Intern with password-reset rights | Keep, limited to standard users, time-limited, manager attestation |

**What I learned**

- Departed contractors and terminated employees were the highest risk: enabled accounts, and in two cases logins after the end date. Those need an investigation, not just a disable.
- Movers (Jordan) and temporary access (Sam, Ruth, Ian) are the usual source of permission creep. Access that was granted for a reason needs an end date.
- Every decision needs a specific rationale and a named owner. "Active employee, access matches role" is not enough.
- My first pass had gaps (blank owners, vague rationales, Ruth marked Keep). I corrected them after a second look.

Remediation (making these changes in AD, with before and after group exports) is next.

---

## Problems I hit and how I fixed them

- **RSAT would not install on the client (`0x800f0950`).** The client could not pull Features-on-Demand payloads from Windows Update. I ruled out a WSUS redirect and edition restrictions, and it resolved after a full Windows Update cycle. The fallback I documented is installing from the Features-on-Demand ISO with `-Source` / `-LimitAccess`. I also learned RSAT is only a remote GUI; AD can be fully managed from the DC with PowerShell.
- **Proxmox thin-pool exhaustion paused my VMs (I/O errors).** The LVM-thin pool (about 49 GB on a 120 GB SSD) was oversubscribed by thin-provisioned disks and snapshots that included RAM state. I diagnosed it with `lvs` / `vgs`, stabilized it by clearing orphaned snapshot volumes and extending the pool, then fixed the root cause by adding a 256GB NVMe drive, creating a new thin pool, and migrating the VM disks. The old pool went from 100% to about 20% usage on the new one.
- **RAM allocation.** Giving a VM 14 GB on a 16 GB host stopped it from booting. I settled on DC 4 GB and client 8 GB, leaving about 2 GB for the host.
- **ADUC rejected my login.** I was signed in with a local account, which has no standing in the domain. Domain administration needs a domain account (`lab\Administrator`).
- **SSH to the client timed out.** `sshd` was listening on port 22 and the firewall rule was enabled and set to Allow, but its profile was Private only. A domain-joined machine uses the Domain profile, so the rule never applied. I fixed it with `Set-NetFirewallRule -Name OpenSSH-Server-In-TCP -Profile Domain,Private`, and confirmed with `Test-NetConnection` instead of ping, which Windows blocks by default.
- **A user removed from a group could still open the folder.** After removing a test user from `Finance-Users`, `dir` on the share still listed files. AD showed the removal was correct, and a session listing on the DC confirmed the connection was really hers. Windows builds a login's group list when the login is created, so an existing login kept the old access. A brand-new login was denied. Offboarding should also end open sessions, and tests need a fresh login.
- **The client ran out of disk space and Extend Volume was greyed out.** The virtual disk had already been enlarged, but a 917 MB recovery partition sat between `C:` and the free space. I disabled the Windows Recovery Environment, deleted that partition with `diskpart` after confirming its type, extended `C:`, and re-enabled recovery.

---

## Where it stands

| Area | Status |
|---|---|
| Infrastructure: Proxmox, NVMe storage, DC, client, DNS, domain join, RSAT, SSH | ✅ Done |
| Access model: OUs, groups, 40 users, shares, NTFS, GPO | ✅ Done |
| Okta: org, MFA, groups, users, group-based app assignment | ✅ Done |
| Access review: 40 accounts reviewed, findings documented | ✅ Done |
| Remediate findings in AD, before/after evidence | 🔲 Next |
| AD Agent sync from AD to Okta | 🔲 Deferred |
| Architecture diagram (draw.io) | 🔲 Planned |

**About the sync:** I mirrored users into Okta by hand. The Okta AD Agent needs a domain-joined Windows Server host with at least 8 GB of RAM, which doesn't fit alongside the current VMs, so I deferred it. In production, users would come from an HR-driven joiner/mover/leaver workflow and sync automatically.

---

## Skills demonstrated

- Active Directory design and administration (Server Core, PowerShell, RSAT)
- Idempotent bulk provisioning of OUs, groups and users
- Role-based access control and least privilege with NTFS and security groups
- Group Policy
- Okta group-based app assignment and MFA enrollment
- Access verification: allow, deny, removal and re-add testing
- Access reviews: joiner/mover/leaver risks, privileged access, decision owners and rationale
- Proxmox administration: VMs, LVM-thin storage, disk migration, resource allocation
- Troubleshooting from symptoms to root cause (see above)

---

## Repository structure

```
.
├── README.md
├── scripts/
│   └── Build-HarborviewAD.ps1     # creates OUs, groups, 40 users, memberships
├── screenshots/                   # access tests, Okta dashboards
└── docs/
    ├── Harborview_Access_Review.xlsx   # completed review of 40 accounts
    └── architecture.png                # (planned)
```

All people and the company in the dataset are fictional.

---
