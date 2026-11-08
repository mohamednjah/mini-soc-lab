# Cloud node: communication design

How the GCP node talks to the rest of the lab, which side starts each connection, and which control stands
in the way at each layer. Background and rejected alternatives are in [decisions.md](decisions.md) and
[network-plan.md](network-plan.md). This is a design: nothing in it is built until the build notes say so.

## Principle

**The cloud side never accepts an inbound connection from the internet, except one narrow administration
path.** Everything the node does for the lab is a connection the node itself starts. The firewalls are
stateful, so replies to an allowed outbound connection need no inbound rule.

## Parties

| Party | Where | Address | Role |
|---|---|---|---|
| Wazuh manager | Lab, VMware `VMnet8` | `10.10.10.20` and tunnel `100.73.251.78` (`tag:manager`) | Receives agent events and enrolments |
| Cloud VM | GCP project `mini-soc-lab`, `lab-vpc` / `lab-subnet` (`europe-west8`) | `10.20.0.0/24`, plus a tunnel address (`tag:cloud-agent`) | Wazuh agent plus Tailscale |
| Admin laptop | Lab host | `10.10.10.1` on `VMnet8` | Administers both |
| Tailscale | Internet | Coordination and relay servers | Introduces the two nodes |
| Google IAP | Internet | `35.235.240.0/20` | Gatekeeper for SSH to the cloud VM |
| Cloud Logging + Pub/Sub | GCP | Google APIs over HTTPS | Carries cloud audit logs |

## Overview

```
                      Tailscale coordination / relay
                       ▲                         ▲
            (2) outbound HTTPS/UDP      (2) outbound HTTPS/UDP
                       │                         │
 ┌─────────── Lab ─────┴──────┐         ┌────────┴───── GCP lab-vpc ─────┐
 │ Wazuh manager              │◀═══(1)══│ Cloud VM (Wazuh agent)         │
 │ tag:manager                │ tunnel  │ tag:cloud-agent  tag iap-ssh   │
 │  10.10.10.20 / 100.73.251.78│ TCP    │  10.20.0.0/24                  │
 └────────▲───────────▲───────┘ 1514   └──────────▲──────────────────────┘
          │           │         1515               │
     (5) admin    (4) pull                     (3) SSH via IAP
   over VMnet8   Pub/Sub  ◀── Cloud Audit Logs      │
          │           │                              │
          └─── admin laptop ─────────────────────────┘
```

## Flows

| # | Flow | Started by | Direction at the receiver | Port |
|---|---|---|---|---|
| 1 | Agent events and enrolment: cloud VM to manager, through the tunnel | Cloud VM | Inbound to the manager | TCP 1514 (events), 1515 (enrolment) |
| 2 | Tailscale control and relay | Both nodes | Outbound only | HTTPS 443, UDP for direct paths |
| 3 | Administration: laptop to cloud VM | Google IAP, for the laptop | Inbound to the cloud VM | TCP 22, from `35.235.240.0/20` only |
| 4 | Cloud audit logs: manager pulls from Pub/Sub | Manager | Outbound only | HTTPS 443 to Google APIs |
| 5 | Administration: laptop to manager | Laptop | Inbound to the manager | Unchanged, over `VMnet8` |

There is no flow in which the manager connects to the cloud VM. A TCP agent connection is bidirectional once
the agent has opened it, so the manager can still send commands to the agent over that same connection.

## Controls on each flow

| Flow | Tailscale policy | Cloud VPC firewall | Host firewall | Notes |
|---|---|---|---|---|
| 1 | `tag:cloud-agent` may reach `tag:manager` on TCP 1514 and 1515 only | Egress allowed by default; it sees only the encrypted tunnel traffic, never port 1514 | Manager: `tailnet` zone allows only 1514 and 1515, with ICMP echo blocked. VM: default zone `public` | Built and working; see the build notes for what the tests do and do not prove |
| 2 | n/a | Egress allowed by default | n/a | Nothing inbound is needed for it |
| 3 | n/a | `allow-iap-ssh`: TCP 22 from the IAP range, to VMs tagged `iap-ssh` | VM's own SSH authentication | IAP checks the Google login first |
| 4 | n/a | n/a | Manager already allows outbound | Needs a credential, see below |
| 5 | n/a | n/a | Manager: `public` zone, lab network only | Not exposed through the tunnel |

## Addressing and naming

- The agent enrols against the manager's **tunnel address, `100.73.251.78`**, not a name. Tailscale's DNS is
  deliberately off on the manager (`--accept-dns=false`), so names would not resolve. Key expiry is disabled on
  the manager, so the address stays valid.
- The cloud VM's **hostname becomes the agent name** and appears in every alert. It is chosen before
  enrolment, since renaming afterwards would orphan history, as happened with the Windows client. Proposed:
  `gcp-node01`.
- The agent version must not be newer than the manager's (4.14.9), so it is installed from the same
  repository version.

## Enrolment sequence

1. Create the VM in `lab-subnet` with the `iap-ssh` tag. Connect through IAP.
2. Install Tailscale and join as `tag:cloud-agent` (an auth key assigns the tag; an interactive login would
   attach the VM to a person instead of a role).
3. Confirm the VM can reach `100.73.251.78` on 1514 and 1515, and nothing else.
4. Install the Wazuh agent with the manager address set to `100.73.251.78` and the chosen hostname.
5. Confirm the agent appears as active in the dashboard.

## Secrets and where they live

None of these goes into the repository; `.gitignore` already covers the usual filenames.

| Secret | Lives on | Needed for | Handling |
|---|---|---|---|
| Tailscale auth key | Typed once on the VM | Joining as `tag:cloud-agent` | One-time use, short expiry |
| Enrolment password (if used) | Manager and the install command | Registering the agent | See decision 1 |
| Service-account key (JSON) | The manager | The Wazuh GCP module reading Pub/Sub | Wazuh's module requires a credentials file. Read-only role, root-only permissions, deleted when the trial ends |

The service-account key is the one place the earlier "no key files" guardrail has to bend. It is for the
manager's module, not the VM, and it is limited to subscribing to one Pub/Sub topic.

## Failure modes

- **Tunnel down:** the agent keeps retrying; the manager marks it disconnected. Check key expiry first.
- **IAP unavailable or permission missing:** administration fails; the VM is otherwise unaffected.
- **Trial ends (1 February 2027):** the VM stops and its resources are deleted unless the account is
  upgraded. The agent disappears from the dashboard, which is the expected end state, not an incident.
- **Spoofed enrolment:** only a node permitted by the Tailscale policy can reach port 1515 at all.

## Verification plan

Each flow gets a test that can fail:

1. From the VM, 1514 and 1515 on the manager connect; 22, 443 and 55000 do not.
2. With the Tailscale policy temporarily widened to allow all ports from the VM, 22 and 443 are still refused,
   which isolates the `firewalld` layer; then the policy is restored. *Done; see the Phase 3 build notes.*
3. IAP SSH works for a tagged VM and fails for an untagged one.
4. The agent shows as active, and a marker event from the VM is searchable in the dashboard.
5. A Cloud Audit Log event appears in Wazuh through the Pub/Sub module.

## Decisions still open

1. **Require a password for agent enrolment?** *Decided: yes.* Enrolment on 1515 needs no password by
   default. The Tailscale policy already limits who can reach it, so the exposure is small, but a password
   closes the gap if the tag is ever misused. It applies to new enrolments only: the agents already
   enrolled on the Windows hosts keep their keys.
2. **Cloud audit logs now or later?** *Decided: later.* Flow 4 adds a service-account key and a logging
   sink, so it is a separate step after the VM and agent work.
3. **VM image and size.** *Checked, to be confirmed.* The public `rocky-linux-cloud` project publishes Rocky
   Linux 10 (family `rocky-linux-10`, plus `rocky-linux-10-optimized-gcp`, a gVNIC variant, and arm64
   families that do not fit `e2`), dated 26 October 2026, alongside 9 and 8. In `europe-west8-a`, `e2-micro`
   (1 GB), `e2-small` (2 GB) and `e2-medium` (4 GB) are all offered. Proposed: `rocky-linux-10` on `e2-small`.
4. **Hostname.** *Decided: `gcp-node01`.* It becomes the agent name in every alert, so it is fixed before
   enrolment.
