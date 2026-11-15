# Network plan

The lab runs on a single isolated IPv4 subnet behind VMware's NAT. Everything on-premises uses
RFC 1918 private addressing; the only public IP in the whole project is the cloud VM added in
Phase 3.

## Topology

```
                        ┌──────────────────────────┐
                        │  NAT gateway             │
                        │  10.10.10.2 → internet   │
                        └────────────┬─────────────┘
                                     │
┌────────────────────────────────────┼──────────────────────────────────┐
│  Lab network — 10.10.10.0/24                                          │
│  VMnet8 (NAT) · DHCP disabled · all addresses static                  │
│                                                                       │
│  ┌────────────────────┐  ┌────────────────────┐  ┌──────────────────┐ │
│  │ Domain controller  │  │ Wazuh manager      │  │ Windows client   │ │
│  │ 10.10.10.10        │  │ 10.10.10.20        │  │ 10.10.10.30      │ │
│  │ Server 2022        │  │ Rocky Linux 10.2   │  │ Windows 10 Pro   │ │
│  │ AD DS + DNS        │  │ SIEM / XDR / SCA   │  │ Domain-joined    │ │
│  └─────────┬──────────┘  └────────────────────┘  └────────┬─────────┘ │
│            │                                              │           │
│            └──────────────── DNS queries ─────────────────┘           │
└───────────────────────────────────────────────────────────────────────┘
```

## Addressing

| Address | Host | Notes |
|---|---|---|
| `10.10.10.1` | Host machine (VMware virtual adapter) | Reserved by VMware. How you SSH/RDP in from Windows. |
| `10.10.10.2` | NAT gateway | Reserved by VMware. Default gateway for all VMs. |
| `10.10.10.10` | Domain controller | Windows Server 2022, AD DS + DNS for `soclab.local` |
| `10.10.10.20` | Wazuh manager | Rocky Linux 10.2 — manager, indexer, dashboard |
| `10.10.10.30` | Windows client | Domain-joined workstation |
| `10.10.10.40` | *reserved* | Kali attacker VM (Phase 4) |
| `10.10.10.50` | *reserved* | Spare / second client |
| `10.10.10.254` | DHCP (disabled) | Reserved by VMware convention; not in use |

Subnet mask `255.255.255.0` · gateway `10.10.10.2` on every host.

## Why these choices

**NAT, not bridged or host-only.** Bridged would put the lab on the home LAN as peers — the DC would
compete with the home router and lab traffic could reach real devices. Host-only has no internet, which
blocks `dnf`, Windows Update, and downloading Wazuh/Sysmon/Atomic Red Team. NAT gives outbound internet
while keeping the lab in its own broadcast domain that the home LAN cannot initiate connections into.

**`10.10.10.0/24`, not `192.168.x.x`.** Home routers almost always use `192.168.0.0/24` or
`192.168.1.0/24`; overlapping with the host's own LAN causes routing confusion that is hard to diagnose.
`10.10.10.0/24` is memorable and collision-free. It also deliberately avoids the cloud default ranges (AWS's default VPC uses `172.31.0.0/16`; Azure's
default VNet uses `10.0.0.0/16`), so a hybrid link never has to deal with overlapping subnets. Tailscale's
own addresses come from `100.64.0.0/10`, which overlaps with none of these.

**DHCP disabled, static everywhere.** Two reasons: servers should not move, and VMware's DHCP hands out
VMware's DNS server — which breaks Active Directory domain joins (see below).

## DNS — the rule that breaks AD if ignored

Active Directory is built on DNS: clients locate the domain through SRV records
(`_ldap._tcp.dc._msdcs.soclab.local`) that only the DC serves. A domain-joined machine pointed at a
public resolver fails to join with errors that never mention DNS.

- **Windows client** → DNS = `10.10.10.10` only. Never `8.8.8.8`, never the NAT gateway.
- **Domain controller** → DNS = itself, plus a **forwarder** to `10.10.10.2` (or `1.1.1.1`) so it can
  still resolve public names for updates.
- **Rocky Linux** → `10.10.10.10` if you want lab name resolution; agents reach the manager by IP anyway.

## Firewall ports (Phase 2)

`firewalld` on Rocky blocks these by default — open them when installing Wazuh:

| Port | Purpose |
|---|---|
| 1514/tcp | Agent → manager event traffic |
| 1515/tcp | Agent enrollment / registration |
| 443/tcp | Wazuh dashboard (web UI) |
| 55000/tcp | Wazuh API |

Windows hosts need no inbound rules — agents connect **outbound** to the manager.

## Isolation caveat

NAT means the VMs can reach the internet. That is fine for Phase 4, because Atomic Red Team executes
*simulated* techniques, not live malware. If real malware samples are ever detonated in this lab, switch
the VMs to a **LAN segment** (fully isolated, no host or internet connectivity) first.

## Phase 3 — connecting the cloud node

The cloud VM must reach the manager, which sits on a private IP behind VMware NAT and a home router —
two layers of address translation. The cloud provider is Google Cloud (see the decisions log); the connectivity
design below does not depend on it.

**Decision: Tailscale.** Both the manager and the cloud VM make outbound connections to Tailscale's
coordination service and form a WireGuard tunnel between them. Each gets a `100.x.y.z` address, and the
cloud agent enrols against the manager's tunnel address. Nothing is opened inbound at home: no router
rule, no VMware port forward.

Rejected alternatives:

- **Port forwarding 1514/1515** — would need two forwards (router, then VMware NAT), a dynamic-DNS name,
  and only works if the ISP provides a public address rather than carrier-grade NAT. It exposes the
  manager to the internet, and by default Wazuh's enrolment port does not require a password.
- **Plain WireGuard** — a sound option with a smaller attack surface and no third-party dependency, but
  more manual configuration and an open UDP port on the cloud VM.

Tailscale is installed **on the Rocky manager itself**, not on the laptop, so the tunnel terminates where the
service is. This adds a network path into the manager, which is a deliberate change:

- **Account security.** Whoever controls the Tailscale login can enrol a device, so multi-factor
  authentication is enabled at the identity provider.
- **Least-privilege policy.** Tailscale's default lets every device reach every other on every port. The
  policy is restricted so the cloud VM, the internet-facing and less-trusted machine, can reach the manager
  on TCP 1514 and 1515 only, and nothing else (no SSH, no dashboard).
- **Firewall.** `firewalld` rules for 1514/1515 are scoped to the tunnel interface rather than opened to
  every source.

In production, an IPsec site-to-site VPN or a zero-trust access service would be the usual pattern; a
home lab cannot terminate a site-to-site tunnel, which is why a mesh VPN is used here.
