# GitHub Issue Conventions (shared reference)

All cross-functional agents (developer, test-engineer, code-reviewer, security-auditor, requirements-analyst, devops-engineer) log their day-to-day work as GitHub issues via the `gh` CLI. The project-manager agent owns milestones; these agents attach their issues to the right milestone but never create or close milestones.

## Filing an issue
```bash
gh issue create \
  --title "<concise, specific title>" \
  --body "<what, where, repro/impact, suggested next step>" \
  --label "<role-label>,<type-label>" \
  --milestone "<exact milestone title, if one applies>"
```

## Standard labels
- Type: `bug`, `enhancement`, `task`, `security`, `tech-debt`, `question`, `change-request` (a proposed change to G1-approved requirements; see `.claude/SDLC_CONVENTIONS.md` §4)
- Role (who raised it): `from:dev`, `from:qa`, `from:review`, `from:security`, `from:requirements`, `from:ops` (post-release / production problems raised by the devops-engineer)

## Trace IDs in issues
For full-lane stories, put the affected IDs (`<KEY>-AC-nn`, `<KEY>-T-nn`, etc.) in the issue body under a `Trace:` line, e.g. `Trace: PWD-AC-02, PWD-T-01`. That's how a defect is linked back to the requirement it breaks in `docs/traceability/`.

## Before filing
- Search first to avoid duplicates: `gh issue list --search "<keywords>" --state all`
- If a matching issue exists, comment on it instead: `gh issue comment <number> --body "..."`

## Rules for all agents
- File issues only within your role's concern (see each agent's own instructions).
- Only attach a milestone if you know the exact title — list them with `gh issue list` context or ask the project-manager agent. Do not invent milestone names.
- Never create, edit, or close milestones — that's the project-manager's job.
- Write issue bodies that a teammate could act on without you: what, where (file/line/endpoint), how to reproduce or why it matters, and a suggested next step.
- Never fabricate an issue number or claim an issue was filed without actually running the command and reporting the resulting URL.
