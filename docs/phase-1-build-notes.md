# Phase 1 — Base infrastructure build notes

Build log for the local lab: virtual network, domain controller, Windows client, and Rocky Linux
host. Addressing and design rationale live in [network-plan.md](network-plan.md); this file records
what was actually done, what broke, and how it was verified.

## Environment

| | |
|---|---|
| Host | Windows 11, i5-11xxxH (6C/12T), 16 GB RAM, 512 GB SSD |
| Hypervisor | VMware Workstation Pro |
| Lab network | VMnet8 (NAT), `10.10.10.0/24` |

RAM is the binding constraint: 2 GB (DC) + 4 GB (client) + 6 GB (Rocky) = 12 GB, leaving ~4 GB for
the host. VMs are powered on per phase rather than all at once.

## Virtual network

VMnet8 reconfigured in **Virtual Network Editor** (requires *Change Settings* for admin rights):

- Subnet IP `10.10.10.0`, mask `255.255.255.0`
- NAT type, host virtual adapter connected (`VMware Network Adapter VMnet8` = `10.10.10.1`)
- Local DHCP service disabled — VMs are statically addressed

![VMnet8 configuration](images/vmnet8-config.png)

Each VM's adapter is set to **Custom: Specific virtual network → VMnet8 (NAT)** rather than the
generic "NAT" radio button, so the target network is explicit in the VM config.

![VM network adapter setting](images/vm-network-adapter.png)

## Static addressing

Configured from an elevated PowerShell session on each Windows host. The `Remove-*` lines make the
sequence idempotent — safe to re-run whether the interface is fresh, on APIPA, or already configured.

```powershell
$if = "Ethernet0"

Set-NetIPInterface -InterfaceAlias $if -Dhcp Disabled

Remove-NetIPAddress -InterfaceAlias $if -AddressFamily IPv4 -Confirm:$false -ErrorAction SilentlyContinue
Remove-NetRoute -InterfaceAlias $if -DestinationPrefix 0.0.0.0/0 -Confirm:$false -ErrorAction SilentlyContinue

New-NetIPAddress -InterfaceAlias $if -IPAddress <address> -PrefixLength 24 -DefaultGateway 10.10.10.2

Set-DnsClientServerAddress -InterfaceAlias $if -ServerAddresses 10.10.10.10
```

**Domain controller — `10.10.10.10`**, DNS pointed at itself.

![DC static IP configuration](images/dc-static-ip.png)

**Windows 10 client — `10.10.10.30`**, DNS pointed at the DC.

![Client static IP configuration](images/client-static-ip.png)

`AddressState: Tentative` in both outputs is Duplicate Address Detection in progress; it settles to
`Preferred` within a couple of seconds.

## Rocky Linux host — `10.10.10.20`

**Rocky Linux 10.2, Minimal ISO** (2.57 GB). Minimal rather than DVD (14.15 GB) because this box is a
headless server: both images install without needing network access, but Minimal skips a package
library that would go unused — everything the host needs comes from network repos over NAT.

Install choices:

| Setting | Value |
|---|---|
| Software selection | Minimal Install |
| Time & date | Europe/Rome |
| Partitioning | Automatic |
| KDUMP | Disabled — reclaims reserved RAM on a memory-constrained VM |
| Hostname | `wazuh-manager` |
| Root SSH login | Disabled; admin access via a `wheel`-group user |
| FIPS mode | Not enabled — see below |

Networking was configured during installation rather than post-install. Note the interface is
**`ens160`** (VMware's name on Linux, not `Ethernet0`), and VMware assigned the paravirtualised
**VMXNET3** adapter automatically.

![Rocky network configuration](images/rocky-network-config.png)

> The screenshot captures the initial entry of `10.10.10.10` only; `10.10.10.2` was added alongside it
> before the install completed. Final configuration is both, in that order.

### DNS: domain controller first, NAT gateway as fallback

Final configuration is **`10.10.10.10, 10.10.10.2`** — the DC first, the NAT gateway second.

The DC alone was rejected. RAM constraints mean only the VMs a given phase needs are powered on, so the
DC is frequently down; with it as sole resolver, `dnf` and the Wazuh installer fail to resolve anything
and surface as timeouts that give no hint a powered-off VM is the cause.

The gateway alone was also viable — this host is not domain-joined and Wazuh agents reach it by IP, so
it has no functional need to resolve `soclab.local` names. Keeping the DC first was chosen anyway, for
the convenience of addressing lab hosts by name.

**Accepted cost:** glibc's resolver walks nameservers in order with a ~5 s timeout, so while the DC is
powered off every lookup stalls before falling through to the gateway. If that becomes irritating,
NetworkManager can shorten it rather than requiring the list to change:

```bash
nmcli con mod ens160 ipv4.dns-options "timeout:1,attempts:1"
```

Note this arrangement is specific to a **non-domain-joined** host. The Windows client must use the DC
and *only* the DC, because a fallback resolver silently breaks the SRV lookups AD clients depend on to
locate the domain. If this host is ever joined via SSSD/realmd, the same restriction starts applying to
it too.

### FIPS mode

Installed in normal mode. FIPS 140-2/140-3 is a US federal cryptographic standard; none of the 23 job
descriptions this lab targets mention it, and Italian/EU frameworks (GDPR, NIS 2, ISO 27001) do not
require it. It also restricts algorithms in ways that commonly break SSH access, package tooling, and
Java-based stacks — OpenSearch under FIPS needs a certified JCE provider and extra JVM configuration,
which would obstruct the Wazuh install for no benefit here.

FIPS can be enabled post-install (`fips-mode-setup --enable`) if a future role ever calls for it, ideally
from a snapshot so the fallout can be documented and rolled back.

### Version note

Planning documents were written against Rocky Linux 9; the build uses **10.2**. Verified compatible —
Wazuh's package list covers *"RockyLinux 10 and later"* (currently `wazuh-agent-4.14.7`).

### Verified configuration

Address and interface state — `10.10.10.20/24` on `ens160`, link `UP`:

```bash
ip -4 addr show ens160
```

![Rocky IP address](images/rocky-ip-addr.png)

Routing — default route via the NAT gateway, plus the directly-connected lab subnet:

```bash
ip route
```

![Rocky routing table](images/rocky-ip-route.png)

Resolver — both nameservers present in the order configured, written by NetworkManager:

```bash
cat /etc/resolv.conf
```

![Rocky resolv.conf](images/rocky-resolv-conf.png)

Note `/etc/resolv.conf` is **generated by NetworkManager**; editing it directly does not persist across
reboots or connection restarts. Resolver changes belong in `nmcli con mod ens160 ipv4.dns ...`.

## What broke

### VMware Easy Install vs. the Windows Server evaluation ISO

**Symptom:** Setup failed immediately with *"Windows cannot find the Microsoft Software License
Terms. Make sure the installation sources are valid and restart the installation."*

**Cause:** Pointing the New Virtual Machine wizard at the ISO silently enables VMware's *Easy
Install*, which generates an `autounattend.xml` answer file and attaches it via a hidden virtual
floppy drive. The generated file specifies a product key and edition index that don't exist in an
evaluation ISO (which ships four editions and expects no key), so Setup cannot resolve the matching
licence terms.

**Ruled out first:** ISO integrity — the file was a complete 5,044,094,976 bytes, matching the
published size, so the download was not truncated.

**Fix:** Remove the **Floppy** device from VM Settings → Hardware, which deletes the answer file from
the VM. For subsequent VMs, avoid the problem entirely by choosing **"I will install the operating
system later"** in the New Virtual Machine wizard, then attaching the ISO to the CD/DVD device
afterwards.

**Applies to:** every VM in this lab, Rocky Linux included — Easy Install causes comparable problems
on Linux guests.

### Windows Server password complexity at first logon

**Symptom:** The initial Administrator password was rejected as not meeting complexity requirements.

**Cause:** The default policy requires three of four character classes (upper, lower, digit, symbol),
a minimum length, and rejects passwords containing the account name.

**Worth noting:** the installer's keyboard layout may not match the physical keyboard, so symbols can
land on different keys than expected. The eye icon on the password field confirms what was actually
typed.

## Verification

From the Windows client, with the DC powered on:

```powershell
Get-NetIPConfiguration                                  # address, gateway, DNS
Get-DnsClientServerAddress -InterfaceAlias Ethernet0 -AddressFamily IPv4
Test-NetConnection 10.10.10.10 -InformationLevel Quiet  # lab connectivity
Test-NetConnection 1.1.1.1 -InformationLevel Quiet      # NAT to internet
```

Expected: static addresses with `PrefixOrigin: Manual`, gateway `10.10.10.2`, DNS `10.10.10.10` only,
and both connection tests returning `True`.

## Outstanding

- [ ] Disable the local DHCP service on VMnet8 — currently still enabled, which contradicts the
      network plan. Not breaking anything while every host is static, but any future VM (or a reset
      adapter) would receive VMware's DNS server and fail to join the domain.
- [ ] Confirm the client's DNS is `10.10.10.10` and nothing else — no public resolver as a secondary.
- [ ] Promote the DC to AD DS, create the `soclab.local` domain, and add a DNS forwarder to
      `10.10.10.2` so the DC can still resolve public names.
- [x] Build the Rocky Linux host at `10.10.10.20`.
- [ ] Join the client to the domain and confirm the computer object appears in AD Users and Computers.
- [ ] On the Rocky host: `dnf update`, install `open-vm-tools`, enable `vmtoolsd`, confirm SSH access
      from the host, then snapshot as a clean baseline before Phase 2.

## Known limitations

- **Single-filesystem layout.** Automatic partitioning puts everything on one root filesystem, so
  several CIS Benchmark controls covering separate `/var`, `/var/log`, `/var/tmp`, `/home` and `/tmp`
  partitions cannot be satisfied without rebuilding the VM. Accepted knowingly: manual LVM layout was
  judged not worth the complexity before the SIEM was even installed, at the cost of a few points on
  the Phase 5 compliance score.
- **Flat network.** All hosts share one broadcast domain with no segmentation between the domain
  controller, the workstation and the SIEM. A production network would separate these into VLANs with
  firewall policy between them, which is what constrains lateral movement after a workstation is
  compromised — so Phase 4 attacks will move laterally more easily here than they would in a real
  environment.
