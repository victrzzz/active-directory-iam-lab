# Active Directory + Okta IAM Lab

A self-hosted Identity and Access Management lab built to demonstrate core IAM analyst skills: directory design, role-based access control, scripted user provisioning, hybrid cloud identity, and access review / recertification.

Built on a local Proxmox hypervisor with a Windows Server 2022 domain controller, a Windows 11 domain-joined client, and Okta (Integrator Free Plan) for cloud app access, then audited with a simulated quarterly access review.

> **Status:** In progress. This README is maintained as the lab is built. See the [Build Status](#build-status) table for what is complete and what is planned.

---

## Why this project

IAM analyst work is fundamentally about *who should have access to what, and proving it stays that way*. This lab recreates that end to end:

- Stand up an on-prem directory (Active Directory) with a role-based access model
- Provision a realistic user population with a repeatable script
- Extend identity to the cloud (Okta) with app-to-group assignment
- Run a **simulated access review** against that population to find and remediate excess access
- Document findings the way a real access recertification would

The access review is the centerpiece. It mirrors day-to-day IAM work rather than just "I set up Active Directory."

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
                    │   Okta (cloud)    │  app-to-group access assignment
                    │ Integrator Free   │  (group-based app access)
                    └───────────────────┘
```

*(A polished architecture diagram, drawn in draw.io, will replace this ASCII sketch: see `docs/architecture.png`.)*

**Domain:** `lab.local`
**Design choice:** The DC runs **Server Core** (no desktop GUI) for a smaller footprint and attack surface, the production-preferred posture. Directory administration is done remotely from the Windows 11 client via **RSAT**, mirroring how admins manage DCs from an admin workstation rather than logging into the DC directly.

---

## Build status

| Phase | Item | Status |
|---|---|---|
| **Infrastructure** | Proxmox host + networking | ✅ Done |
| | VM storage moved to 256GB NVMe (LVM-thin) | ✅ Done |
| | Server 2022 Core promoted to DC (`lab.local`) | ✅ Done |
| | DNS configured (DC authoritative + forwarder) | ✅ Done |
| | Windows 11 client: static IP, DNS pointed at DC | ✅ Done |
| | Client domain-joined to `lab.local` | ✅ Done |
| | RSAT installed on client (ADUC / GPMC / DNS) | ✅ Done |
| | OpenSSH Server on client, reachable from admin PC | ✅ Done |
| **Access model** | OU structure (Finance / HR / IT / Security Groups) | ✅ Done |
| | Security groups (20, role-based) | ✅ Done |
| | 40 users provisioned by script, with group memberships | ✅ Done |
| | Shared folders + NTFS permissions on groups (Finance by hand, HR and IT scripted) | ✅ Done |
| | Verify group-based access (allow, deny, removal) | ✅ Done |
| | GPO linked to Finance OU: maps the Finance share as `F:` | ✅ Done |
| **Cloud identity** | Okta Integrator Free Plan org, MFA (Okta Verify) enrolled | ✅ Done |
| | Groups and 6 users mirrored by hand from AD | ✅ Done |
| | Apps assigned to the Finance group; access verified per user | ✅ Done |
| | AD Agent sync (AD to Okta) | 🔲 Deferred (needs an 8 GB member server) |
| **Access review** | Simulated review dataset (40 accounts) built | ✅ Done |
| | Perform review: decision + rationale per account | 🔲 In progress |
| | Export live group membership from AD for comparison | 🔲 Planned |
| | Remediate in AD + document before/after | 🔲 Planned |
| **Docs** | Architecture diagram (draw.io) | 🔲 Planned |
| | Findings table + writeup | 🔲 Planned |

---

## Access model (design)

Access is assigned **to groups, never to individuals**: role-based, the way real IAM work is done.

**Organizational Units (by department):** Finance · HR · IT, plus a Security Groups OU.
OUs organize objects and scope policy and delegation. Permissions attach to groups, not OUs.

**Security groups:** global security groups per department and role, for example `Finance-Users`, `Finance-Managers`, `Payroll-Processors`, `HR-Users`, `HRIS-Admins`, `IT-Helpdesk`, `Server-Admins`.

**Users:** 40 fictional users for a fictional company ("Harborview Logistics"), spread across Finance, HR and IT with realistic job titles and role-appropriate group membership. Created with [`scripts/Build-HarborviewAD.ps1`](scripts/Build-HarborviewAD.ps1), which is safe to re-run: it skips OUs, groups and users that already exist.

**Shared folders:** `\\DC01\Finance`, `\\DC01\HR` and `\\DC01\IT`. Each share is open to Everyone at the share level, and NTFS permissions do the gatekeeping: inheritance is removed, and only Administrators, SYSTEM and that department's `-Users` group (Modify) have access. Access follows group membership rather than being granted per person. Finance was built by hand (Computer Management for the share, the Security tab for NTFS) to learn the process; HR and IT were created with a script.

**Group Policy:** a GPO named `Finance - Map Share` is linked to the Finance OU and maps `\\DC01\Finance` as `F:` at logon. The OU link decides who receives the policy; the `Finance-Users` group still decides who can actually open the folder. A Finance user sees and can open the drive; an HR user does not get it.

**Okta:** `Finance-Users`, `HR-Users` and `IT-Users` groups and six users (two per department) were created by hand in an Okta Integrator Free Plan org, mirroring AD. Apps are assigned to the Finance group only: a Finance user sees them on the Okta dashboard after MFA enrollment, and an HR user sees none. Okta has no OUs, so access is driven by groups; in a synced setup the AD Agent would import users from selected OUs and the groups themselves.

> In production, AD to Okta would sync automatically via the **Okta AD Agent** with delegated authentication, and users would be created by an HR-driven joiner/mover/leaver workflow rather than a script. In this lab the script and manual mirroring stand in for that, because the goal is to demonstrate the *access model and review*, not directory-sync plumbing. The AD Agent needs a domain-joined Windows Server host with at least 8 GB of RAM, which does not fit alongside the current VMs, so sync is deferred until there is room.

---

## The access review (centerpiece)

A simulated quarterly access recertification across the 40 accounts:

1. **Review** each account's job, manager, employment type, last login, systems and group membership against its business justification.
2. **Decide** per account: Keep, Modify access, Remove access, Disable account, or Escalate, and record the access to remove, the decision owner, and a written rationale.
3. **Compare** the decisions against live group membership exported from AD (`Get-ADGroupMember | Export-Csv`).
4. **Remediate** in AD and document before and after in a findings table (`docs/access-review.md`).

The dataset contains a mix of the problems access reviews exist to catch: people who changed roles and kept old access, expired contractors, standard-role users holding admin rights, dormant accounts, and accounts that do not match HR status. It also contains legitimately elevated users who should be kept, because over-flagging is its own mistake.

Differentiated access is verified by logging into the client as different domain users sequentially (for example, a Finance user cannot reach HR's share).

---

<!-- TEMP-START: working notes, delete this whole section when docs/access-review.md is written -->
## Access verification (working notes)

Tested with one Finance user (`premington`) against the department shares:

| Test | Expected | Result |
|---|---|---|
| Member of `Finance-Users` opens `\\DC01\Finance` | Allowed | ✅ Allowed |
| Same user opens `\\DC01\HR` | Denied | ✅ Access is denied |
| Removed from `Finance-Users`, fresh login opens `\\DC01\Finance` | Denied | ✅ Access is denied |
| Added back to `Finance-Users`, fresh login | Allowed | ✅ Allowed |
| Finance user signs in at the console | `F:` drive appears and opens | ✅ Appears and opens |
| HR user signs in at the console | No `F:` drive | ✅ No `F:` |
| Finance user signs in to Okta | Assigned apps appear | ✅ Two apps appear |
| HR user signs in to Okta | No apps | ✅ Empty dashboard |

Note: `net use` succeeds against these shares regardless of the user, because the share itself is open to Everyone. The check that matters is opening the folder, which is where NTFS applies.
<!-- TEMP-END -->

---

## Skills demonstrated

- Active Directory design & administration (Server Core, PowerShell-first)
- Scripted, idempotent bulk provisioning of OUs, groups and users
- DNS fundamentals in a domain context (authoritative zone + forwarders)
- Role-Based Access Control (group-based permissions, least privilege)
- Access review / recertification and remediation
- NTFS / share permission management (planned)
- Hybrid identity: on-prem AD and Okta (planned)
- Hypervisor administration (Proxmox: VMs, LVM-thin storage, disk migration, resource allocation)
- Real-world troubleshooting (see below)

---

## Challenges & resolutions

Real build, real problems. Documenting these shows troubleshooting ability, not just a clean tutorial run.

- **RSAT would not install on the client (`0x800f0950`).** The Windows 11 Enterprise *Evaluation* client could not pull Features-on-Demand payloads from Windows Update (ruled out a WSUS redirect and edition restrictions as causes). Resolved after a full Windows Update cycle restored FoD access; the documented fallback is installing from the Features-on-Demand ISO with `-Source` / `-LimitAccess`. Takeaway: RSAT is only a remote GUI. AD can be fully managed from the DC via PowerShell regardless.
- **Proxmox thin-pool exhaustion paused VMs (I/O errors).** The host's LVM-thin pool (~49 GB on a 120 GB SSD) was oversubscribed by thin-provisioned VM disks and snapshots that included RAM state. Diagnosed with `lvs` / `vgs`, stabilized by clearing orphaned snapshot volumes and extending the pool into unallocated space. Root cause was an undersized SSD, so I added a 256GB NVMe drive, created a new LVM-thin pool, and migrated the VM disks to it; the original pool dropped from 100% to about 20%. Takeaway: thin provisioning hides the real ceiling, and snapshots with RAM state are expensive.
- **RAM allocation vs. host headroom.** Over-allocating a VM (14 GB on a 16 GB host) prevented it from starting. Settled on DC 4 GB + client 8 GB, leaving about 2 GB for the host, with the SIEM VM powered off during IAM work.
- **ADUC rejected the login ("user name or password is incorrect").** The client session was signed in with a local account, which has no standing in the domain. Domain administration requires signing in as a domain account (`lab\Administrator`).
- **SSH to the domain-joined client timed out.** `sshd` was running and listening on port 22, and the `OpenSSH-Server-In-TCP` firewall rule was enabled and set to Allow, yet connections still failed. The rule's profile was **Private** only, while a domain-joined machine sits on the **Domain** profile, so the rule never applied. Fixed with `Set-NetFirewallRule -Name OpenSSH-Server-In-TCP -Profile Domain,Private`. Takeaway: an enabled Allow rule can still not apply, so check the profile it covers, and test the actual port with `Test-NetConnection` rather than relying on ping, which Windows blocks by default.
- **A user removed from a group could still open the folder.** After removing a test user from `Finance-Users`, `dir` on the share still listed files. AD showed the removal was correct, and a session listing on the DC confirmed the user's connection was real. Windows builds a login's group list when the login is created, so a login that already existed kept the old access. A brand-new login (a fresh SSH session as that user) was denied, as expected. Takeaway: removing someone from a group does not end sessions they already have open, so offboarding should also end their sessions, and tests must use a fresh login.
- **Client was nearly out of disk space, and Extend Volume was greyed out.** The Proxmox virtual disk had already been enlarged, so 20 GB showed as unallocated in Disk Management, but a 917 MB recovery partition sat between `C:` and the free space, and Windows can only extend into adjacent space. Fixed by disabling the Windows Recovery Environment (`reagentc /disable`), deleting the recovery partition with `diskpart` after confirming its type with `list partition`, extending `C:`, and re-enabling recovery (`reagentc /enable`). Takeaway: resizing the hypervisor disk and growing the guest partition are two separate steps, and partition order matters.

---

## Lab methodology

Each component is built to *learn it*, not just complete it:

1. Follow a guide once to get a working result
2. Rebuild from memory (for example, OUs, groups and membership created by hand in ADUC and again in PowerShell before running the bulk script)
3. Deliberately misconfigure and troubleshoot
4. Document what broke and what would be done differently

---

## Repository structure

```
.
├── README.md                      # this file
├── docs/
│   ├── architecture.png           # draw.io architecture diagram
│   ├── access-review.md           # findings table + before/after
│   └── setup-notes.md             # build steps / commands used
├── scripts/
│   └── Build-HarborviewAD.ps1     # creates OUs, groups, 40 users, memberships
├── data/
│   └── Harborview_Access_Review.xlsx   # simulated review dataset (fictional)
└── screenshots/                   # ADUC, Okta, access-review evidence
```

All people and the company in the dataset are fictional.

---

*Part of a broader cybersecurity portfolio. See [github.com/victrzzz](https://github.com/victrzzz). Built by Victor Alonso Gaspar.*
