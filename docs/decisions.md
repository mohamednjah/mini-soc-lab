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

## 2026-11-08 — Wazuh's cloud reader gets one permission on one subscription
- **Status:** Accepted, not yet proven to work
- **Context:** The Wazuh GCP module needs a service-account key file on the manager to pull audit logs from
  Pub/Sub, which is the one exception to the "no key files" guardrail. Wazuh's credentials guide grants the
  account Pub/Sub Publisher and Subscriber, with no stated scope. The module only reads.
- **Decision:** Create a dedicated service account with no roles, and grant Subscriber on the single
  subscription the module reads. The sink's own writer identity separately gets Publisher on the topic only.
- **Alternatives considered:** following Wazuh's guide as written, at project level (simplest, but a stolen
  key could then also publish, and the scope would be the whole project); Workload Identity Federation to
  avoid a key file entirely (it needs an external identity provider, which is out of proportion for a lab).
- **Consequences:** A stolen key can read audit logs, which name the account and every admin action, and
  nothing else. It still has to be treated like a password: root-only permissions on the manager, never in
  the repository, deleted when the trial ends. If the module needs more than Subscriber on that subscription,
  the extra permission is added and this entry updated, not edited in place.

## 2026-11-05 — Block ICMP echo in the manager's tunnel zone
- **Status:** Accepted
- **Context:** The `tailnet` zone was meant to admit only TCP 1514 and 1515. A plain `ping` from the cloud
  VM to the manager's tunnel address was expected to fail and instead succeeded 14 times out of 14.
  `firewalld` allows ICMP unless a type is blocked, and the zone blocked none. The access policy, whose rule
  is TCP only, also let it through, which is not yet explained.
- **Decision:** Add `echo-request` to the zone's ICMP blocks, permanently.
- **Alternatives considered:** leaving ICMP allowed (harmless in practice, but it makes the stated
  "two ports only" untrue); adding an ICMP rule to the Tailscale policy so ping is deliberately allowed
  (nothing needs it).
- **Consequences:** The zone now matches its description. `tailscale ping` still works for path diagnosis.
  A side effect is the first independent proof that the host layer can deny traffic without the policy,
  for ICMP only. The question of why the policy passed ICMP is open.

## 2026-11-05 — Cloud VM host firewall: default zone from `trusted` to `public`
- **Status:** Accepted
- **Context:** On inspection the Rocky 10 image's `eth0` was in `firewalld`'s `trusted` zone, which accepts
  everything. The VM has an ephemeral external IP, so the VPC firewall (one ingress rule: IAP on TCP 22) was
  the only filter facing the internet. The manager has several layers; the less-trusted cloud node had one.
- **Decision:** Make `public` the default zone, after reading it to confirm it allows SSH
  (`cockpit dhcpv6-client ssh`) so IAP access would survive.
- **Alternatives considered:** leaving `trusted` and relying on the VPC and the Tailscale policy (a
  legitimate choice, but a single misconfigured VPC rule would then expose every port, and the planned
  misconfiguration demo does that deliberately); a custom zone with only SSH (more precise, more to maintain).
- **Consequences:** Defence in depth on the cloud node. The zone allows SSH from any source, so the IAP
  range restriction still depends on the VPC rule. `cockpit` remains allowed but unused. The change is
  permanent, so rollback needs a live session. It might have altered the Tailscale path (direct to
  relayed), which was not measured beforehand.

## 2026-11-04 — Require a password for Wazuh agent enrolment
- **Status:** Accepted
- **Context:** Wazuh's enrolment service on TCP 1515 accepts any agent that can reach it, with no
  password, by default. The cloud node's agent will reach it over the tunnel, so port 1515 is the one
  surface through which a rogue agent could register and feed false events into the SIEM.
- **Decision:** Enable password authentication on the manager (`use_password` in the `<auth>` section and
  a root-owned `authd.pass`), and supply the password when enrolling the cloud agent.
- **Alternatives considered:** leaving it open and relying on the Tailscale policy alone, which already
  restricts 1515 to the cloud-agent tag. Reasonable for a lab, but it leaves a single control between a
  misused tag and a poisoned SIEM.
- **Consequences:** One more secret to handle: it lives only on the manager and in the install command,
  never in the repository. It affects new enrolments only; the two Windows agents already hold keys.
  A forgotten password blocks re-enrolment of a rebuilt VM until it is reset.

## 2026-11-04 — Administering the cloud VM: Identity-Aware Proxy tunnel
- **Status:** Accepted
- **Context:** The cloud network has no inbound rules, which is the intent, but a VM that nothing can reach
  has to be set up somehow. The data path is Tailscale; this is about the administration path.
- **Decision:** One ingress rule allowing TCP 22 only from Google's IAP address range (`35.235.240.0/20`),
  applied only to VMs carrying the network tag `iap-ssh`. Access is by `gcloud compute ssh --tunnel-through-iap`.
- **Alternatives considered:**
  - **A startup script that joins Tailscale with a one-time key** — no inbound rule at all, but the key sits
    in the VM's metadata for a while, and administration afterwards still needs a route.
  - **SSH from the internet, or from the home IP** — contradicts the guardrail of exposing nothing on 22;
    the home address is also dynamic, so the rule would keep breaking.
- **Consequences:** Nothing on the internet can reach SSH: only Google's proxy can, and it checks a Google
  login first. The cost is a single, narrow exception to "no ingress rules", plus a dependence on Google's
  proxy and on the account having permission to use the tunnel. Any VM that should be reachable has to be
  given the tag deliberately, and a VM without it stays closed.

## 2026-11-02 — Cloud provider: Google Cloud, replacing AWS
- **Status:** Accepted. Supersedes *Cloud provider: AWS over Azure* (2026-10-21) and its clarification below.
- **Context:** The AWS choice rested on its Free plan, which closes itself when credits run out and so
  limits spending. After signing up, AWS showed that this account is **not eligible for the free plan**:
  the registration details matched an existing or previously registered AWS account, free plans being
  exclusive to new customers. The offered alternative was a paid plan with no credit and pay-as-you-go
  billing from the first charge. Until then the Billing console showed "Unable to load" for every cost
  figure and an empty credit balance, consistent with an account left waiting on that decision. The AWS
  paid-plan prompt was never confirmed. Eligibility was a condition of the free plan that was not checked
  before choosing AWS.
- **Decision:** Google Cloud, on its Free Trial.
- **Alternatives considered:**
  - **AWS paid plan** — would work, but means real spending with no credit buffer, for a project whose cloud
    node needs a few hours of VM time. Declined.
  - **Azure** — the original plan. Not tried: its credit lasts only 30 days and keeping anything running after
    that requires upgrading to pay-as-you-go, and its free account has the same new-customer eligibility rule.
  - **Evading the eligibility check with a second account** — against AWS's terms, and risks the account.
- **Consequences:** The account shows €269 of credit (the $300 welcome credit), none used, expiring
  **1 February 2027**. Google states there are no automatic charges, so spending starts only if the account is
  upgraded by hand; the Upgrade button is therefore never used. When the trial ends, resources stop, and are
  permanently deleted if the account is not upgraded during the grace period, so the cloud node has a hard
  deadline roughly 90 days out. GCP is named alongside AWS and Azure in two of the job descriptions, and Wazuh
  has a GCP module (Pub/Sub and Storage). GCP-specific guardrails are listed in the Phase 3 notes, including
  checking the default network for firewall rules that allow SSH or RDP from anywhere.

## 2026-11-02 — AWS Free plan: some actions upgrade it automatically (clarifies the AWS decision)
- **Status:** Accepted. Qualifies a claim in the 2026-10-21 AWS entry.
- **Context:** That entry says overspending is impossible "without deliberately upgrading to a Paid plan".
  AWS's own documentation says a free plan **automatically upgrades** to a paid plan if the account joins AWS
  Organizations, sets up a Control Tower landing zone, joins the AWS Partner Network, signs a Professional
  Services contract or Enterprise Agreement, buys a Skill Builder Team subscription, or is designated HIPAA-
  or SEC-compliant.
- **Decision:** Keep AWS, and treat the spending guarantee as conditional: it holds only while none of those
  actions is taken. Use a plain IAM user for day-to-day access and avoid anything that creates an AWS
  Organization. Whether other services, such as IAM Identity Center, create one on a free plan has not been
  checked, so they are avoided too.
- **Consequences:** The cost guardrail is no longer purely a property of the account; it depends on the
  account staying free of those triggers, so the budget alert is still needed as a backstop.

## 2026-10-21 — Cloud provider: AWS over Azure
- **Status:** Accepted. Supersedes the earlier working assumption of Azure in the Phase 3 plan.
- **Context:** The plan assumed Azure, chosen for its fit with a Windows and Active Directory lab, the
  Microsoft-heavy job market, and the SC-200 certificate. Before committing, the current free-tier terms
  were read from each provider's own pages rather than relied on from memory.
- **Decision:** AWS, on its Free plan.
- **Alternatives considered:**
  - **Azure** — $200 of credit for the first 30 days only, then a required upgrade to pay-as-you-go to keep
    anything running, after which usage beyond the free quantities is billed. Microsoft may also remove
    compute resources from an account inactive for 30 days, which suits a lab used in bursts poorly.
- **Consequences:** AWS's Free plan credits ($100 on signup, up to $100 more for completing activities)
  last up to six months, and the account closes itself when they run out or the period ends, so overspending
  is not possible without deliberately upgrading to a Paid plan. That makes the cost guardrail a property of
  the account instead of an alert. The costs of the choice: a hard six-month lifetime for the cloud node, and
  a different certification path (AWS Cloud Practitioner rather than SC-200 or AZ-900). The job descriptions
  name AWS and Azure equally, so the market gave no tiebreak, and the earlier claim that the Italian market
  favours Microsoft was an assumption the data did not support.

## 2026-10-21 — Cloud-to-manager connectivity: Tailscale
- **Status:** Accepted
- **Context:** The Phase 3 cloud VM must send agent traffic to a manager on `10.10.10.20`, which sits behind
  VMware NAT and a home router. The cloud has a public address; the manager has none.
- **Decision:** Tailscale, installed on the Rocky manager and the cloud VM, with a restrictive access policy
  and multi-factor authentication on the account.
- **Alternatives considered:**
  - **Port forwarding 1514/1515** — needs forwards on both the router and VMware NAT, a dynamic-DNS name,
    and a public address the ISP may not provide (carrier-grade NAT). It exposes the manager to the
    internet, and Wazuh's enrolment port needs no password by default. Rejected on security grounds.
  - **Plain WireGuard** — smaller attack surface, no third party, and easy to automate in Phase 6, but
    manual key handling, no identity or audit layer, and an open UDP port on the cloud VM.
- **Consequences:** Nothing opens inbound at home and the setup works behind any ISP. The cost is a
  dependency on a third-party coordination service, and a new weak point: the Tailscale account. Tailscale's
  default policy allows every device to reach every other, so the policy must be tightened so the cloud VM
  can reach only TCP 1514 and 1515 on the manager. In production, a site-to-site IPsec VPN or zero-trust
  access service would be expected instead; this is a home-lab compromise, and it should be described as one.

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
