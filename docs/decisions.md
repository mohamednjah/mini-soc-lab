# Mini-SOC — Decisions Log

Append-only record of the *why* behind the build. Newest entry at the top.
Each entry is a real decision (a fork with alternatives), not a task — "installed Wazuh" is a
task; "chose Wazuh over ELK" is a decision.

Never edit or delete a past entry. If a decision changes, add a **new** entry with
`Status: Superseded by [YYYY-MM-DD entry title]` or `Status: Reversed`, and say why the original
call didn't hold up. The reversals are as valuable as the original decisions.

Format:

```markdown
## YYYY-MM-DD — <short decision title>
- **Status:** Accepted | Superseded by [...] | Reversed
- **Context:** what problem/constraint forced a choice
- **Decision:** what I chose
- **Alternatives considered:** X (why not), Y (why not)
- **Consequences:** what this makes easy / what it costs or defers
```

---

## 2026-09-03 — Repo location: standalone folder, not inside Resume-Creator
- **Status:** Accepted
- **Context:** The lab needs its own public GitHub repo so it can be linked from the CV and
  LinkedIn independently of the private CV-tailoring repo.
- **Decision:** `mini-soc-lab` lives in its own folder (`Desktop/mini-soc`) with its own git
  history, separate from the Resume-Creator repo that tracks job applications.
- **Alternatives considered:** a subfolder inside Resume-Creator — rejected because it would mix
  an eventually-public lab repo with a private CV/job-application repo, and tie their git
  histories together for no benefit.
- **Consequences:** Clean separation; the lab repo can be made public without exposing anything
  from the CV-tailoring work.

## 2026-09-03 — Scope of Phase 0: full scaffold + git up front
- **Status:** Accepted
- **Context:** Could either build structure incrementally per-phase, or set up the full folder
  layout, README, and decisions log now.
- **Decision:** Full scaffold now (all folders, README with planned architecture, this decisions
  log, .gitignore, git init + first commit).
- **Alternatives considered:** minimal-first (just README + decisions.md, add folders as needed)
  — rejected because the README's repo-layout section is more useful as a map from day one, and
  empty folders cost nothing to create.
- **Consequences:** Every later phase drops files into a folder that already exists and is
  already explained in the README, instead of restructuring mid-build.
