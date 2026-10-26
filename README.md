# Active Directory + Okta IAM Lab

A self-hosted Identity and Access Management lab built to demonstrate core IAM analyst skills: directory design, role-based access control, hybrid cloud identity, and access review / recertification.

Built on a local Proxmox hypervisor with a Windows Server 2022 domain controller, a Windows 11 domain-joined client, and Okta (free tier) for cloud app access — then audited with a mock quarterly access review.

> **Status:** In progress. This README is maintained as the lab is built — see the [Build Status](#build-status) table for what's complete vs. planned.

---

## Why this project

IAM analyst work is fundamentally about *who should have access to what, and proving it stays that way*. This lab recreates that end to end:

- Stand up an on-prem directory (Active Directory) with a role-based access model
- Extend identity to the cloud (Okta) with app-to-group assignment
- Deliberately introduce **access sprawl**, then run a **mock access review** to find and remediate it
- Document findings the way a real access recertification would

The access review is the centerpiece — it's the part that mirrors day-to-day IAM work rather than just "I set up Active Directory."

---

## Architecture

```
                     Proxmox Host (HP EliteDesk 800 G2 Mini)
                     Intel i5-6500T · 16GB RAM
  ┌──────────────────────────────────────────────────────────────┐
  │                                                                │
  │   ┌─────────────────────────┐      ┌────────────────────────┐ │
  │   │  DC01  (VM 101)          │      │  Client  (VM 102)      │ │
  │   │  Windows Server 2022     │◄────►│  Windows 11 Enterprise │ │
  │   │  Server Core (no GUI)    │ domain│  Domain-joined         │ │
  │   │  Domain Controller       │ join  │  RSAT (ADUC/GPMC/DNS)  │ │
  │   │  DNS for lab.local       │      │  Admin workstation     │ │
  │   │  192.168.1.160           │      │  192.168.1.161         │ │
  │   └─────────────────────────┘      └────────────────────────┘ │
  │                                                                │
  └──────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │   Okta (cloud)    │  app-to-group access assignment
                    │   free dev tier   │  (hybrid identity)
                    └───────────────────┘
```

*(A polished architecture diagram — drawn in draw.io — will replace this ASCII sketch: see `docs/architecture.png`.)*

**Domain:** `lab.local`
**Design choice:** The DC runs **Server Core** (no desktop GUI) for a smaller footprint and attack surface — the production-preferred posture. All directory administration is done remotely from the Windows 11 client via **RSAT**, mirroring how admins manage DCs from an admin workstation rather than logging into the DC directly.

---

## Build status

| Phase | Item | Status |
|---|---|---|
| **Infrastructure** | Proxmox host + networking | ✅ Done |
| | Server 2022 Core promoted to DC (`lab.local`) | ✅ Done |
| | DNS configured (DC authoritative + forwarder) | ✅ Done |
| | Windows 11 client — static IP, DNS → DC | ✅ Done |
| | Client domain-joined to `lab.local` | ✅ Done |
| | RSAT installed on client (ADUC / GPMC / DNS) | ✅ Done |
| **Access model** | Security groups created | ✅ Done |
| | OU structure (Finance / IT / HR / Sales) | 🔲 In progress |
| | Users (2–3 per dept + manager) | 🔲 In progress |
| | Shared folders + NTFS perms on groups | 🔲 Planned |
| | One GPO applied (e.g. password policy) | 🔲 Planned |
| **Cloud identity** | Okta free tier — users mirrored | 🔲 Planned |
| | App integration assigned to groups only | 🔲 Planned |
| **Access review** | Introduce deliberate access sprawl | 🔲 Planned |
| | Run mock quarterly review + findings table | 🔲 Planned |
| | Remediate + document before/after | 🔲 Planned |
| **Docs** | Architecture diagram (draw.io) | 🔲 Planned |
| | Findings table + writeup | 🔲 Planned |

---

## Access model (design)

Access is assigned **to groups, never to individuals** — role-based, the way real IAM work is done.

**Organizational Units (by department):** Finance · IT · HR · Sales

**Security groups (per department):** e.g. `Finance-Users`, `Finance-Managers`

**Shared folders:** `\\DC01\Finance`, `\\DC01\HR`, etc. — NTFS permissions bound to the department groups, so access follows group membership rather than being granted per person.

**Okta:** a subset of AD users mirrored into Okta; at least one app integration (sample SAML app) assigned to specific groups only — demonstrating app-to-group assignment, not just on-prem AD.

> In production, AD → Okta would sync automatically via the **Okta AD Agent** with delegated authentication (AD remains the credential source of truth). In this lab the users are mirrored manually — the goal is to demonstrate the *access model and review*, not directory-sync plumbing.

---

## The access review (centerpiece)

The capstone of the lab — a mock quarterly access recertification:

1. **Introduce sprawl deliberately** — place a user in a group they shouldn't be in; leave a "terminated" account still enabled.
2. **Review** — export group memberships and evaluate who should/shouldn't have access, and why.
3. **Remediate** — revoke excess access, disable the stale account.
4. **Document** — before/after, in a findings table (`docs/access-review.md`).

Differentiated access is verified by logging into the client as different domain users sequentially (e.g. a Finance user cannot reach HR's share).

---

## Skills demonstrated

- Active Directory design & administration (Server Core, PowerShell-first)
- DNS fundamentals in a domain context (authoritative zone + forwarders)
- Role-Based Access Control (group-based permissions, least privilege)
- NTFS / share permission management
- Hybrid identity (on-prem AD ↔ Okta cloud)
- Access review / recertification and remediation
- Hypervisor administration (Proxmox: VMs, storage, resource allocation)
- Real-world troubleshooting (see below)

---

## Challenges & resolutions

Real build, real problems — documenting these shows troubleshooting ability, not just a clean tutorial run.

- **RSAT would not install on the client (`0x800f0950`).** The Windows 11 Enterprise *Evaluation* client could not pull Features-on-Demand payloads from Windows Update (ruled out WSUS redirect and edition restrictions as causes). Resolved after a full Windows Update cycle restored FoD access; the documented fallback is installing from the Windows 11 Features-on-Demand ISO with `-Source` / `-LimitAccess`. Key takeaway: RSAT is only a remote GUI — AD can be fully managed from the DC via PowerShell regardless.
- **Proxmox thin-pool exhaustion paused VMs (I/O errors).** The host's LVM-thin pool (~49 GB on a 120 GB SSD) was oversubscribed by thin-provisioned VM disks and snapshots-with-RAM-state. Diagnosed via `lvs` / `vgs`, recovered by clearing orphaned snapshot/state volumes and extending the pool into unallocated space. Root cause: undersized SSD — resolved with a storage upgrade. Takeaway: thin provisioning hides the real ceiling; snapshots including RAM state are expensive.
- **RAM allocation vs. host headroom.** Over-allocating a VM (14 GB on a 16 GB host) prevented it from starting. Settled on DC 4 GB + client 8 GB, leaving ~2 GB for the host; ELK powered off during IAM work.

---

## Lab methodology

Each component is built to *learn it*, not just complete it:

1. Follow a guide once to get a working result
2. Rebuild from memory
3. Deliberately misconfigure and troubleshoot
4. Document what broke and what would be done differently

---

## Repository structure

```
.
├── README.md                 # this file
├── docs/
│   ├── architecture.png      # draw.io architecture diagram
│   ├── access-review.md      # findings table + before/after
│   └── setup-notes.md        # build steps / commands used
├── scripts/                  # PowerShell used to build the environment
└── screenshots/              # ADUC, Okta, access-review evidence
```

---

*Part of a broader cybersecurity portfolio — see [github.com/victor-gaspar](https://github.com/victor-gaspar). Built by Victor Alonso Gaspar.*
