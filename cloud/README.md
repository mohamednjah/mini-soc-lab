# cloud/

Configuration, screenshots, and write-up for the GCP hybrid node (Phase 3): VPC firewall rules, IAM
least-privilege setup, the Cloud-Audit-Logs-to-Wazuh integration, and the deliberate
misconfiguration-then-fix demo. Also holds the versioned Tailscale access policy.

No secrets, keys, or credentials belong in this folder — see `.gitignore` at the repo root.

- `tailscale-policy.json`: the versioned Tailscale access policy
- `wazuh/`: the Wazuh GCP module configuration (an example, no secrets) and the custom rule 100500 for the
  firewall misconfiguration demo

The build and the reasoning are in [../docs/phase-3-build-notes.md](../docs/phase-3-build-notes.md).
