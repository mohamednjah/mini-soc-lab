# Mini-SOC: a detection-engineering lab

A home lab built to practise what security teams actually do: collect logs from endpoints and the cloud,
detect suspicious changes, harden systems and measure the result. It was designed from the gaps that kept
recurring in real cybersecurity and infrastructure job descriptions: SIEM, cloud, hardening, EDR/XDR,
Active Directory, automation and incident response.

**The documentation is the deliverable.** Every claim here points at a note, a decision, a rule or a
screenshot. The notes include the mistakes and the things I could not verify, on purpose.

## Where it stands

| Phase | What | Status |
|---|---|---|
| 0 | Repository scaffold, decisions log, hypervisor choice | Done |
| 1 | Isolated network, Active Directory domain `soclab.local`, domain-joined client | Done: [notes](docs/phase-1-build-notes.md) |
| 2 | Wazuh 4.14.9 SIEM/XDR on Rocky Linux 10.2, Sysmon, auditd, Windows Security log, CIS baselines | Done: [notes](docs/phase-2-build-notes.md) |
| 3 | Hybrid cloud node on Google Cloud, joined to the lab over Tailscale; cloud audit logs into Wazuh; a deliberate misconfiguration, detected and fixed | Done: [notes](docs/phase-3-build-notes.md) |
| 4 | Detection engineering: simulated attacks, Sigma rules, detection cards, incident write-ups | Next |
| 5 | CIS hardening with before/after compliance scores | Planned |
| 6 | Ansible: rebuild the lab from code | Planned |
| 7 | Python/Pandas alert analytics and anomaly detection | Planned |
| 8 | Final write-up, limitations, next steps | Planned |

## What exists today

- **An Active Directory lab.** A Windows Server 2022 domain controller and a domain-joined Windows 10 client
  on an isolated VMware NAT network with static addressing and the DNS ordering Active Directory needs.
- **A working SIEM.** Wazuh (manager, indexer, dashboard) on Rocky Linux 10.2, a platform outside Wazuh's
  recommended list, with the reasoning and the check recorded. Three log sources are verified end to end
  with traceable test events: Sysmon process telemetry, the Windows Security log and Linux auditd.
- **Compliance baselines before any hardening.** CIS scores from Wazuh: domain controller 27%, Windows
  client 28%, cloud node 55% ([hardening/baseline.md](hardening/baseline.md)). Phase 5 will measure the change.
- **A cloud node reporting into the on-premises manager.** A Rocky Linux VM on Google Cloud enrols with a
  password and sends events through a Tailscale (WireGuard) tunnel. Nothing is opened inbound at home, and
  the VM is administered only through Google's Identity-Aware Proxy.
- **Layered controls.** A Tailscale access policy and a `firewalld` zone on the tunnel interface, tested
  separately: the policy was widened temporarily to show the host firewall holds on its own. A cloud VPC
  firewall admits SSH only from Google's proxy range; its negative test (a direct connection must time out)
  is still on the open list.
- **Cloud audit logs as a detection source.** Google Cloud Audit Logs flow through a log sink and Pub/Sub into
  Wazuh. A custom rule ([100500](cloud/wazuh/100500-gcp-firewall-open-to-internet.xml)) fires when a firewall rule
  is opened to the whole internet. The demo created, detected and fixed such a rule, using a disabled
  rule that targets no machine so nothing was ever exposed
  ([detection card](detections/cards/gcp-firewall-rule-open-to-internet.md)).
- **A decisions log with reversals.** [18 entries](docs/decisions.md), including the ones that did not hold up:
  the move from AWS to Google Cloud, and corrections to claims I had made too confidently.

## Architecture

```
+-Home lab: VMware NAT 10.10.10.0/24, static addresses ------------------+
|                                                                        |
| DC01             WIN10-CLIENT      Wazuh manager                       |
| Server 2022      Windows 10 Pro    Rocky Linux 10.2                    |
| AD DS + DNS      domain-joined     indexer, server, dashboard          |
| Sysmon + agent   Sysmon + agent    alerts, auditd                      |
| 10.10.10.10      10.10.10.30       10.10.10.20                         |
|      |                 |                  ^                            |
|      +--- agent events-+------------------+                            |
|                                                                        |
+------------------------------------------------------------------------+
   gcp-node01 --> manager: agent events, TCP 1514/1515, inside a
                  Tailscale tunnel (WireGuard)
   manager --> Google: pulls audit logs from Pub/Sub, outbound HTTPS
+-Google Cloud, free trial: project network lab-vpc, europe-west8 -------+
|                                                                        |
| gcp-node01                    Cloud Audit Logs -> log sink ->          |
| Rocky Linux 10, Wazuh agent   Pub/Sub topic (Admin Activity entries)   |
| Admin access: Identity-Aware Proxy only, no inbound rule from internet |
|                                                                        |
+------------------------------------------------------------------------+
```

Design documents: [network plan](docs/network-plan.md) and the
[cloud node's communication design](docs/communication-design.md).

## What I learned

- **Test the assumption.** Several of my own expectations were wrong when I checked them: the cloud VM's host
  firewall was wide open by default, a TCP-only access rule still let ping through, and a project name in my
  own notes did not exist. Each is recorded where it happened, with the correction.
- **A layered defence is only proven when each layer is tested alone.** The first port test passed because
  two controls blocked it at once, so it could not show that either worked. Widening one layer on purpose
  separated them.
- **A platform guardrail is information.** Google's organization policy refused to issue a service-account
  key. I overrode it for one project for a few minutes and wrote down why, rather than treating it as an
  obstacle, and weighed a keyless design as the alternative.
- **Ask what a test cannot show.** Most sections of the notes end with what was not verified.

## Limitations and open items

- **No attack simulation yet.** The only custom detection so far is a cloud one, tested through the live
  pipeline once. I make no claim about detection rates until Phase 4.
- **The cloud node is temporary.** It runs on a free trial that ends on 1 February 2027, after which its
  resources are deleted. The evidence is the notes and the rule, not a running VM.
- **Some checks are not captured.** Where I confirmed a step but did not record it in a screenshot or
  output, the notes say so.
- **Open questions:** whether the tunnel is relayed because of my double NAT or
  because of a firewall change, and Google's acceptable-use policy, which I have not read in full.
- **Sources.** The cloud design document ends with a table of which claims were checked against official
  documentation, which were confirmed only by testing, and which one rested on reasoning alone.
- **One secret exists.** A read-only service-account key on the manager, with its handling and the
  rationale in the decisions log. It is revoked at the end of the trial.
- **One laptop.** Everything runs on a single machine, and the VMs are powered down when idle.

## Repository layout

```
docs/             Decision log, network plan, design notes, per-phase build notes, screenshots
cloud/            Tailscale access policy; Wazuh GCP module example and custom rule 100500 (Phase 3)
detections/
  cards/          One card per detection: ATT&CK ID, log source, logic, test, false positives, tuning
  sigma/          Sigma rules (Phase 4)
hardening/        CIS baseline scores now; controls applied and after-scores later (Phase 5)
incidents/        Incident write-ups: timeline, evidence, remediation (Phase 4)
infra/            Ansible playbooks and inventory (Phase 6)
analytics/        Python/Pandas pipeline and model evaluation (Phase 7)
```

Nothing secret belongs in this repository: keys, passwords, tokens and credentials are kept out by
`.gitignore`, and screenshots have account names, project identifiers and secrets redacted.
