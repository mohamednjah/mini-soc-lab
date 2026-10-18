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

## Verification still to do

- [ ] Confirm the services are running: `wazuh-manager`, `wazuh-indexer`, `wazuh-dashboard`.
- [ ] Check the indexer heap actually applied: `grep -E "^-Xm[sx]" /etc/wazuh-indexer/jvm.options`.
- [ ] Open the firewall: ports 443, 1514, 1515 and 55000 over TCP.
- [ ] Reach the dashboard at `https://10.10.10.20` from the lab host and log in.
- [ ] Check for SELinux denials (`ausearch -m avc -ts recent`) — it remains enforcing.
- [ ] Snapshot the host once the stack is confirmed healthy.
- [ ] Enrol agents on `DC01` and `WIN10-CLIENT`, then install Sysmon on both.
