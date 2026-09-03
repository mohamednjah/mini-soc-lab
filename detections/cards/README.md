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

*(Empty until Phase 4.)*
