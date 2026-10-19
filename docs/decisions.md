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

## 2026-10-19 — Sysmon configuration: SwiftOnSecurity despite its age
- **Status:** Accepted
- **Context:** Sysmon logs nothing useful without a configuration, and the configuration decides what is
  recorded and what is silently dropped. Windows event logs alone give thin process visibility.
- **Decision:** Use the SwiftOnSecurity `sysmonconfig-export.xml` on both Windows hosts.
- **Alternatives considered:**
  - **`sysmon-modular`** — actively maintained (updated within the week) and organised by MITRE ATT&CK
    technique, a good fit for Phase 4. Not chosen first because I could not confirm a prebuilt-config
    URL, and a heavily commented config is easier to learn from.
  - **Native process-creation auditing (event 4688 with command lines)** — free and needs no extra
    software, but gives no network connections, hashes or process-access events. A partial answer.
- **Consequences:** Well documented and widely used, but last updated July 2024. It loads as schema 4.50
  against Sysmon's 4.91, so newer event types are outside what it knows to log or filter — a blind spot
  to name in any detection coverage claim. Revisit with `sysmon-modular` during Phase 4.

## 2026-10-19 — Elevate per process rather than log on as Domain Admin
- **Status:** Accepted
- **Context:** Installing agents and Sysmon on the workstation needs admin rights, and the only admin
  identity available was the domain Administrator.
- **Decision:** Stay logged on as the standard user `mnjah` and elevate individual PowerShell processes
  with the domain Administrator's credentials through UAC.
- **Alternatives considered:** signing in interactively as `SOCLAB\Administrator` — simpler, but it
  caches Domain Admin credential material in LSASS on the workstation, the condition most credential-theft
  chains depend on.
- **Consequences:** Privileged credentials are used for one process and not left in the session. Elevation
  also leaves explicit-credential and special-privilege logon events (4648, 4672) as a legitimate
  baseline for Phase 4 to distinguish from the malicious equivalent. Costs a credential prompt per task.

## 2026-10-18 — Leave the Wazuh indexer heap at the installer default
- **Status:** Accepted
- **Context:** Planning assumed the OpenSearch heap would need capping on a 6 GB VM. Reading the installer
  showed it derives the heap from a variable that is never defined and falls back to 1 GB, which a
  post-install check confirmed (`-Xms1024m`, `-Xmx1024m`). The risk is an undersized heap, not an
  oversized one.
- **Decision:** Keep 1 GB.
- **Alternatives considered:** raise it to roughly half of RAM (about 3 GB), per Wazuh's tuning guidance.
  Rejected for now: two agents produce little indexed data, and the memory is scarce on a 16 GB host.
- **Consequences:** The dashboard stays responsive with both Windows hosts reporting. This is one
  observation at lab scale; it is the first setting to raise if Phase 4 volume produces slowness or
  indexer circuit-breaker errors.

## 2026-10-18 — Wazuh all-in-one on Rocky 10.2, outside the installer's recommended list
- **Status:** Accepted
- **Context:** The 4.14 installation assistant accepts Rocky but treats only 9.4 as recommended, and the
  VM has 6 GB of RAM against a documented recommendation of 8 GiB. The documentation's recommended-OS
  list includes RHEL 10.
- **Decision:** Install all-in-one (indexer, server and dashboard on one host) on the Rocky 10.2 build,
  accepting the warning.
- **Alternatives considered:**
  - **Wazuh's pre-built virtual appliance** — faster, but opaque, and it would undercut the reason for
    building on a RHEL-family system, which is to do the administration by hand.
  - **Ubuntu** — on the supported list, but it duplicates existing Debian-family experience.
- **Consequences:** An untested rather than unsupported configuration. It installed cleanly, all three
  services run, SELinux stayed enforcing with no denials, and the stack handles two agents. 6 GB sits below
  the recommendation, so the host should be resized if Phase 4 load demands it.

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
