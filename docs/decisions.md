# Mini-SOC — Decisions Log

Append-only record of the *why* behind the build. Newest entry at the top.
Each entry is a real decision (a fork with alternatives), not a task — "installed Wazuh" is a
task; "chose Wazuh over ELK" is a decision.

Never edit or delete a past entry. If a decision changes, add a **new** entry with
`Status: Superseded by [YYYY-MM-DD entry title]` or `Status: Reversed`, and say why the original
call didn't hold up. The reversals are as valuable as the original decisions.

Format:

```markdown
## YYYY-MM-DD — <short decision title>
- **Status:** Accepted | Superseded by [...] | Reversed
- **Context:** what problem/constraint forced a choice
- **Decision:** what I chose
- **Alternatives considered:** X (why not), Y (why not)
- **Consequences:** what this makes easy / what it costs or defers
```

---

## 2026-09-03 — Lab network: VMware NAT on 10.10.10.0/24, static IPs
- **Status:** Accepted
- **Context:** The lab needs internet access (dnf, Windows Update, downloading Wazuh/Sysmon/Atomic
  Red Team) but must not be reachable from — or able to interfere with — the home LAN. It also needs
  a stable addressing scheme that will not collide with Azure when the hybrid node lands in Phase 3.
- **Decision:** Single subnet `10.10.10.0/24` on VMware's NAT network (VMnet8, reconfigured), VMware
  DHCP disabled, all hosts statically addressed. DC at `.10`, Wazuh manager at `.20`, Windows client
  at `.30`; `.40`/`.50` reserved. Full plan in `network-plan.md`.
- **Alternatives considered:**
  - **Bridged networking** — rejected: puts VMs on the home LAN as peers, so the lab DC would compete
    with the home router and lab traffic could reach real household devices.
  - **Host-only networking** — rejected: no internet at all, which blocks package installs and every
    tool download the build depends on. (Still the right mode if live malware is ever detonated —
    noted as a caveat in the network plan.)
  - **`192.168.x.x` addressing** — rejected: overlaps the ranges home routers almost always use.
  - **Leaving VMware DHCP enabled** — rejected: it hands out VMware's DNS server, which breaks
    Active Directory domain joins, and servers should not have moving addresses anyway.
- **Consequences:** Outbound internet without inbound exposure; no subnet overlap with Azure's default
  `10.0.0.0/16` when the Phase 3 link is built. Costs a little manual work — every host is configured
  by hand, and the DC must be given a DNS forwarder so it can still resolve public names.

## 2026-09-03 — Hypervisor: VMware Workstation Pro over VirtualBox
- **Status:** Accepted
- **Context:** Phase 1 needs 3 VMs running concurrently on one Windows host (Domain Controller,
  Windows client, Rocky Linux), plus a Wazuh manager later — snapshot reliability and networking
  flexibility (host-only/internal networks, NAT) matter more here than on a single-VM setup.
- **Decision:** VMware Workstation Pro (free for personal use as of 2024).
- **Alternatives considered:**
  - VirtualBox — free, cross-platform, but generally weaker snapshot performance and less
    reliable virtual networking under multi-VM load than Workstation Pro.
  - Hyper-V — built into Windows at no cost, but conflicts with other hypervisors on the same
    host and has rougher edges for isolated lab networking/AD scenarios.
- **Consequences:** Better snapshot/rollback behaviour for iterating on the AD/Wazuh setup without
  rebuilding from scratch after a mistake. Trade-off: Workstation Pro is Windows/Linux-only
  (irrelevant here, host is Windows) and is a heavier install than VirtualBox.

## 2026-09-03 — Scope of Phase 0: full scaffold + git up front
- **Status:** Accepted
- **Context:** Could either build structure incrementally per-phase, or set up the full folder
  layout, README, and decisions log now.
- **Decision:** Full scaffold now (all folders, README with planned architecture, this decisions
  log, .gitignore, git init + first commit).
- **Alternatives considered:** minimal-first (just README + decisions.md, add folders as needed)
  — rejected because the README's repo-layout section is more useful as a map from day one, and
  empty folders cost nothing to create.
- **Consequences:** Every later phase drops files into a folder that already exists and is
  already explained in the README, instead of restructuring mid-build.
