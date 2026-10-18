# Phase 2 — Wazuh SIEM/XDR build notes

Deployment of the Wazuh manager on the Rocky Linux host (`10.10.10.20`, `wazuh-manager`), followed by
agent enrolment on the Windows hosts. Addressing is in [network-plan.md](network-plan.md); the host
itself is documented in [phase-1-build-notes.md](phase-1-build-notes.md).

## Installation

Wazuh **4.14.9**, deployed all-in-one (indexer, server and dashboard on one host) using the official
installation assistant, run over SSH from the lab host with only the Rocky VM powered on — the install
has no dependency on the Windows VMs, so leaving them off kept RAM free for it.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

![Installer start](images/wazuh-install-start.png)

The installer ran from 23:01:24 to 23:09:32 — about eight minutes — and finished with the dashboard
initialised and the generated `admin` credentials printed.

![Installer completion](images/wazuh-install-complete.png)

The generated password is deliberately redacted in the screenshot above. The installer prints it once
to the terminal, so it is recorded in a local, gitignored credentials file rather than in this
repository. Screenshots of that screen are the one place in this project where a routine documentation
habit would publish a real credential.

### Before running it

The installation assistant was read before being executed, which turned up several things worth
recording:

- **Rocky 10.2 is outside the installer's recommended list.** `check_dist` accepts `rocky` as a
  distribution but treats only 9.4 as recommended; anything else sets a flag that prints a warning and
  continues rather than aborting. The documentation's recommended-OS list covers RHEL 7–10 and CentOS
  Stream 10, so this is an untested configuration rather than an unsupported one.
- **The hardware check passes, but the lab sits below the recommendation.** The script's hard
  minimum is 2 cores and roughly 4 GB of RAM for the all-in-one install. The quickstart documentation
  recommends 4 vCPU and 8 GiB for 1–25 agents. This VM has 4 vCPU and 6 GB, which is adequate for a
  two-agent lab but short of the guidance.
- **The indexer heap is probably fixed at 1 GB.** The script derives the OpenSearch heap from a
  variable (`ram_mb`) that is never defined, so the arithmetic evaluates to zero and falls back to a
  1024 MB default regardless of the VM's memory. Inferred from reading the script, not yet observed —
  see the verification list below.

## Firewall

`firewalld` blocks everything outside its default services, so the Wazuh ports were opened explicitly:

```bash
sudo firewall-cmd --permanent --add-port={443,1514,1515,55000}/tcp && sudo firewall-cmd --reload
```

![Firewall ports opened](images/wazuh-firewall-ports.png)

| Port | Purpose |
|---|---|
| 443/tcp | Dashboard web interface |
| 1514/tcp | Agent event traffic |
| 1515/tcp | Agent enrolment |
| 55000/tcp | Wazuh API |

firewalld keeps *runtime* and *permanent* configuration separately: `--permanent` saves a rule without
activating it, and `--reload` applies the saved set. The two `success` lines are one for each half.
The brace expansion is done by bash, not firewalld, and produces four `--add-port` arguments.

All four ports are inbound to the manager; the Windows agents connect outward and need no rules.
Port 9200 (the indexer) is deliberately left closed, since the dashboard reaches it locally.

Rules were applied to the whole `public` zone, so any host that can reach the manager can reach these
ports. Acceptable on a flat lab network; a production deployment would restrict sources to the agent
subnet with a rich rule.

## Verification

All three services report `active`:

```bash
sudo systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard
```

![Wazuh services active](images/wazuh-services-active.png)

The dashboard is reachable at `https://10.10.10.20` from the lab host and the login succeeds. The
browser flags the connection as not secure because the installer generates a self-signed certificate;
that is expected on an isolated lab network and would be replaced with a CA-issued certificate in a
real deployment.

![Wazuh dashboard overview](images/wazuh-dashboard-overview.png)

This confirms the stack runs on Rocky 10.2 despite the installer's recommended-OS warning, so the
untested-configuration caveat above can be treated as resolved for this version.

The overview shows **no agents registered** — as expected, none have been enrolled — yet already
reports 71 medium and 120 low severity alerts for the last 24 hours. With no agents, these are most
likely the manager monitoring itself, which Wazuh does through a built-in local agent (ID `000`). The
source has not been confirmed. It matters because it is the first baseline of "normal" noise: Phase 4
detections will need to be distinguished from it, and the Phase 5 configuration-assessment findings
will start from whatever the manager already reports about its own host.

### Post-install checks

```bash
sudo grep -E "^-Xm[sx]" /etc/wazuh-indexer/jvm.options
sudo firewall-cmd --list-ports
sudo ausearch -m avc -ts recent
```

![Post-install checks](images/wazuh-post-install-checks.png)

- **Indexer heap is 1024 MB** (`-Xms1024m`, `-Xmx1024m`). This confirms the reading of the installer:
  the heap is derived from a variable that is never defined, so it falls back to a fixed 1 GB whatever
  the VM's memory. The file is root-readable only, hence `sudo` on a plain `grep`.
- **All four ports are live** at runtime, not merely saved to the permanent configuration.
- **No SELinux denials.** The system remained in enforcing mode throughout, and the install produced
  no AVC records.

The heap is left at 1 GB for now. Wazuh's tuning guidance is roughly half of system RAM, which would be
about 3 GB here, but with two agents the indexed volume is small and an unnecessary increase would
take memory from a host that is already short of it. It is the first thing to raise if the dashboard
becomes slow or the indexer logs circuit-breaker errors once Sysmon starts feeding events in.

## Verification still to do

- [x] Confirm the services are running: `wazuh-manager`, `wazuh-indexer`, `wazuh-dashboard`.
- [x] Check the indexer heap actually applied: it is 1024 MB, as predicted from the installer.
- [x] Open the firewall: ports 443, 1514, 1515 and 55000 over TCP.
- [x] Confirm the rules are live: `sudo firewall-cmd --list-ports`.
- [x] Reach the dashboard at `https://10.10.10.20` from the lab host and log in.
- [ ] Identify the source of the 191 alerts already present with no agents enrolled.
- [x] Check for SELinux denials (`ausearch -m avc -ts recent`) — none, with SELinux still enforcing.
- [x] Snapshot the host once the stack is confirmed healthy (`02-wazuh-installed`), giving a rollback
      point with Wazuh installed and no agents enrolled.
- [ ] Enrol agents on `DC01` and `WIN10-CLIENT`, then install Sysmon on both.
