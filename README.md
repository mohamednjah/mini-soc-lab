# Mini-SOC — Detection-Engineering Lab

A home lab built to close recurring gaps found across real Italian cybersecurity/infrastructure job
descriptions:
SIEM, cloud, hardening, EDR/XDR, Active Directory, automation, and incident response.

**The documentation is the deliverable.** Every claim on my CV traces to something in this repo:
a rule, a screenshot, a metric, or a playbook.

## Status

**Phases 1 and 2 complete.** An isolated lab network with a `soclab.local` Active Directory domain and a
domain-joined Windows client (Phase 1), and a Wazuh 4.14.9 manager on Rocky Linux 10.2 receiving events
from agents on both Windows hosts (Phase 2). Three log sources are verified end to end with traceable test
events: Sysmon process telemetry, the Windows Security log, and Linux auditd on the manager. Pre-hardening
CIS compliance baselines are recorded.

**Phase 3 in progress.** A GCP node (`gcp-node01`) reports to the on-prem manager through a Tailscale
tunnel, with an access policy and host firewalls limiting it to the two Wazuh ports. The agent is enrolled
with a password and shows as active. Still to come: the Cloud Audit Logs integration and the deliberate
misconfiguration demo, plus a few open checks listed in the Phase 3 notes.

- [docs/phase-1-build-notes.md](docs/phase-1-build-notes.md) — what was built, what broke, how it was verified
- [docs/phase-2-build-notes.md](docs/phase-2-build-notes.md) — Wazuh deployment, agents, Sysmon, end-to-end checks
- [docs/phase-3-build-notes.md](docs/phase-3-build-notes.md) — Tailscale, GCP network and VM, agent enrolment, firewall layers
- [docs/communication-design.md](docs/communication-design.md) — how the cloud node talks to the lab
- [hardening/baseline.md](hardening/baseline.md) — CIS baseline scores before any hardening
- [docs/network-plan.md](docs/network-plan.md) — addressing and network design
- [docs/decisions.md](docs/decisions.md) — the reasoning behind each choice

## Architecture (planned)

```
                        ┌─────────────────────────┐
                        │   GCP (free trial)      │
                        │   Linux VM (agent)      │
                        │   FW + IAM least-priv   │
                        └────────────┬────────────┘
                                     │ Wazuh agent
                                     │ + cloud audit logs
┌────────────────────────────────────┼────────────────────────────┐
│  Home lab (isolated virtual network)                            │
│                                     │                           │
│  ┌──────────────┐  ┌──────────────┐│┌───────────────┐           │ 
│  │ Windows Server│ │ Windows      │││ Rocky Linux 9 │           │
│  │ + AD DS       │ │ client       │││ (RHEL-family) │           │
│  │ (soclab.local)│ │              │││               │           │
│  └──────┬───────┘  └──────┬───────┘│└──────┬────────┘           │
│         │  Sysmon+agent   │ agent   │        │  agent           │
│         └─────────────────┴─────────┴────────┘                  │
│                            │                                    │
│                   ┌────────▼──────────┐                         │
│                   │  Wazuh manager    │  ← SIEM / XDR / SCA     │
│                   │  (Rocky Linux)    │                         │
│                   └────────┬──────────┘                         │
└────────────────────────────┼────────────────────────────────────┘
                              │ alert export (API)
                     ┌────────▼─────────┐
                     │ Python / Pandas   │  ← anomaly detection,
                     │ + Sktime pipeline │     model evaluation
                     └───────────────────┘
```

*(Diagram will be replaced with real screenshots as each phase lands.)*

## Repo layout

```
infra/            Ansible playbooks + inventory (Phase 6)
detections/
  sigma/          Sigma detection rules (Phase 4)
  cards/          One markdown card per detection: ATT&CK ID, log source,
                  logic, test, false positives, tuning notes (Phase 4)
incidents/        Incident response write-ups: timeline, evidence, remediation (Phase 4)
hardening/        CIS Benchmark controls applied + before/after SCA scores (Phase 5)
analytics/        Python/Pandas pipeline, anomaly detection, model evaluation (Phase 7)
cloud/            Cloud node config, firewall rules, misconfiguration demo (Phase 3)
docs/             Decision log, diagrams, per-phase notes
```

## What I learned

*(filled in as phases complete)*

## Limitations / next steps

*(filled in at Phase 8)*
