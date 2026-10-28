# Phase 3 — Cloud node build notes

Extends the lab with a cloud VM that reports to the on-premises Wazuh manager. The provider is **AWS**
(Free plan) and the link is a **Tailscale** mesh VPN; both choices and the rejected alternatives are in
[decisions.md](decisions.md), and the connectivity design is in [network-plan.md](network-plan.md).

## Tailscale on the manager

Tailscale is installed on the Rocky manager itself, so the tunnel terminates where the service runs,
rather than on the laptop. The repository definition was read before use: HTTPS, with both package and
repository signature checking enabled, and the signing key served from the same host.

```bash
sudo curl -fsSL https://pkgs.tailscale.com/stable/rhel/10/tailscale.repo -o /etc/yum.repos.d/tailscale.repo
sudo dnf install -y tailscale
sudo systemctl enable --now tailscaled
sudo tailscale up --accept-dns=false
```

The repo file was fetched directly rather than through `dnf config-manager`, which depends on a plugin that
a minimal install may not have.

![Tailscale install on the manager](images/tailscale-install-manager.png)

The signing key was imported and the transaction checks passed. Tailscale 1.104.4 installed, along with
`iptables-nft` as a dependency. The login URL printed by `tailscale up` is a one-time link tying the machine
to an account, so it is redacted in the screenshot.

**`--accept-dns=false`.** On Linux, Tailscale by default takes over DNS configuration. That would risk
overwriting the resolver arrangement verified in Phase 1 (the domain controller first, the NAT gateway as
fallback), and this host does not need Tailscale's name resolution.

The manager received the tunnel address `100.73.251.78`. Tailscale's `100.64.0.0/10` range overlaps none of
the lab network, AWS's default VPC range, or Azure's.

![Manager in the Tailscale admin console](images/tailscale-machine-manager.png)

The admin console shows the manager connected. Account email, device ID, node key, tailnet domain and
external endpoints are redacted. The page reports key expiry **six months from now**: untagged devices carry
the default expiry, so without a tag the manager would silently drop off the network after that period.
Tagging it, as described below, is expected to remove that.

## Still to do

- [x] Confirm `/etc/resolv.conf` still lists the two original nameservers after `tailscale up`: it lists
      the domain controller first and the NAT gateway second, so `--accept-dns=false` left DNS alone.
- [ ] Write the access policy (tags `tag:manager` and `tag:cloud-agent`; the cloud VM may reach the manager
      on TCP 1514 and 1515 only) and confirm its built-in tests pass.
- [ ] Re-run `tailscale up` with `--advertise-tags=tag:manager`, then confirm in the console that the tag
      is applied and key expiry is off.
- [ ] Put `tailscale0` in its own `firewalld` zone allowing only TCP 1514 and 1515.
- [ ] Confirm multi-factor authentication is on for the Tailscale identity provider and the AWS root user.
- [ ] AWS guardrails (budget, no public SSH or RDP, testing policy) before any resource is created.
- [ ] Launch the cloud VM, join it to Tailscale as `tag:cloud-agent`, and enrol its Wazuh agent against the
      manager's tunnel address.
- [ ] Ship CloudTrail into Wazuh, then run the deliberate misconfiguration demo.
