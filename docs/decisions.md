# Decision log

Disagreements between roles that were escalated to the human, and what the human decided. The process is in `.claude/SDLC_CONVENTIONS.md` §7. The orchestrator appends entries here; the owning role applies each decision to its artifact.

Newest first. A decision is reopened only with new evidence, as a new entry that references the old one.

| ID | Date | Question | Parties (owner first) | Decision | Decided by | Dissent | Applied in |
|---|---|---|---|---|---|---|---|
<!-- Example row:
| DR-PWD-01 | 2026-11-24 | Store reset tokens hashed or plaintext with short TTL? | architect vs security-auditor | Hashed (SHA-256), keep 15-min TTL (owner's proposed agreement) | Thanh | none | ADR password-reset rev 2; test plan rev 2 |
-->
