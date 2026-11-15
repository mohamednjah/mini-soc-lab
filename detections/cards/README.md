# detections/cards/

One markdown file per detection, documenting it like a working analyst would:

```markdown
# <Detection name>
- ATT&CK: Txxxx (<technique>)
- Log source: <Sysmon EventID / auditd / ...>
- Detection logic: <plain-English summary + link to ../sigma/xxx.yml>
- Test that triggers it: <Atomic Red Team test ID / manual command>
- False positives: <benign activity that looks similar>
- Tuning notes: <what was changed and why>
```

Cards so far:

- [GCP firewall rule created or changed to allow 0.0.0.0/0](gcp-firewall-rule-open-to-internet.md): Phase 3, cloud

*(The Windows and Linux cards arrive in Phase 4.)*
