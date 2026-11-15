# GCP firewall rule created or changed to allow 0.0.0.0/0

- **ATT&CK:** T1562.007, Impair Defenses: Disable or Modify Cloud Firewall (tactic: Defense Evasion).
  Wazuh's own rule for firewall creation maps to the parent technique T1562 only; this rule uses the
  sub-technique.
- **Log source:** Google Cloud Audit Logs, Admin Activity (`cloudaudit.googleapis.com/activity`), delivered
  through a log sink to Pub/Sub and pulled by Wazuh's `gcp-pubsub` module (see
  [../../cloud/wazuh/gcp-pubsub-module.conf.example](../../cloud/wazuh/gcp-pubsub-module.conf.example)).
- **Detection logic:** [../../cloud/wazuh/100500-gcp-firewall-open-to-internet.xml](../../cloud/wazuh/100500-gcp-firewall-open-to-internet.xml),
  Wazuh rule 100500, level 12. It is a child of the `gcp` rule group and fires when
  `protoPayload.methodName` is `v1.compute.firewalls.insert`, `.patch` or `.update` **and** the request's
  `sourceRanges` contain `0.0.0.0/0`. Wazuh's built-in rule 65053 ("GCP firewall rule created", level 3)
  fires on the same event; this rule is the one that says it is dangerous. It is not a Sigma rule, because
  Sigma was not used for this source.
- **Test that triggers it:** create a firewall rule open to the whole internet, as a disabled rule aimed at
  a tag no VM has, so nothing is exposed:
  `gcloud compute firewall-rules create demo-open-rdp --network=lab-vpc --direction=INGRESS --action=ALLOW
  --rules=tcp:3389 --source-ranges=0.0.0.0/0 --target-tags=demo-no-such-tag --disabled`.
  The alert appeared in `/var/ossec/logs/alerts/alerts.json` about two minutes later, through the live
  pipeline (this rule was not tested with `wazuh-logtest`).
- **False positives:**
  - A rule created or edited with `0.0.0.0/0` that is **disabled**, as in the test itself, or that targets no
    machine. The rule does not read `disabled` or `targetTags`.
  - **Deny** rules with `0.0.0.0/0` as the source, which block traffic and are harmless. The rule does not
    check whether the rule allows or denies.
  - Legitimate public services: a rule opening 80 or 443 to the internet on a web server is intended. The
    rule does not read the port.
- **Tuning notes (not yet done):**
  - Require `disabled` to be absent or false, and exclude deny rules, to remove the first two false
    positives.
  - Alert harder on management ports (22, 3389, 5985, 5986) than on 80 and 443.
  - IPv6 (`::/0`) and wide ranges such as `0.0.0.0/1` are not matched.
  - Only one of the two log entries Google writes per operation carries the request, so the rule fires once
    per operation. If that ever changed, alerts would double.
  - Nothing correlates a creation with a later deletion; the fix is only logged by Wazuh's level-3 rule 65054.

## Evidence

The demo ran against the lab's own project on 2026-11-15, using a rule created disabled with the tag
`demo-no-such-tag`, so no traffic was ever allowed. It is a simulation of the mistake, not a real exposure.

| Stage | What was seen |
|---|---|
| Before | `gcloud compute firewall-rules list` on the lab network: only `allow-iap-ssh`, enabled |
| Mistake, before the custom rule existed | `demo-open-ssh` created. Two level-3 alerts, rule 65053 "GCP firewall rule created" (T1562) |
| Mistake, after the custom rule | `demo-open-rdp` created. One alert, rule **100500**, level 12, "GCP firewall rule created or changed to allow 0.0.0.0/0 (open to the internet)", T1562.007 |
| Fix | Both rules deleted. Four level-3 alerts, rule 65054 "GCP firewall rule deleted" (T1562.007), two per rule |
| After | Only `allow-iap-ssh` again, enabled |

Compute writes two audit entries per operation: an insert produced two 65053 alerts and the two deletions
produced four 65054 alerts. The custom rule fired once, which suggests only one of an insert's two entries
carries the request fields; that is an inference.
