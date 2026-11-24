# SDLC Conventions (shared reference)

This file defines the contracts the agents use to hand work to each other: stable IDs, artifact status, the traceability matrix, change requests, feedback routing, the handoff block every agent returns, and how disagreements are settled. It sits beside `.claude/GITHUB_CONVENTIONS.md`. The gates themselves (G1 to G6) and the pipeline order are in `CLAUDE.md`.

The idea comes from heavyweight, gated SDLCs: roles don't depend on shared memory. They depend on **artifacts** that one role produces and another consumes, **gates** that a human approves, and a **trace** that ties every requirement to its design, code, tests and defects. Our agents also share no memory, so this matters even more here than it does for a human team.

## 1. Stable IDs

The `requirements-analyst` assigns IDs when it writes a requirements doc. Once a doc passes G1, its IDs never change meaning. If something is removed, the ID is retired, not reused.

- **Feature key:** a short uppercase tag per requirements doc, e.g. `PWD` for `docs/requirements/password-reset.md`. It must be unique across `docs/requirements/`, so check the existing docs first.
- **User story:** `<KEY>-US-<nn>`, e.g. `PWD-US-01`
- **Acceptance criterion:** `<KEY>-AC-<nn>`, e.g. `PWD-AC-03`. **The AC is the unit of traceability.** Every downstream artifact points at AC IDs.
- **Non-functional requirement:** `<KEY>-NFR-<nn>`, e.g. `PWD-NFR-01` for "P95 latency under 300ms". These are traced like ACs.
- **Design decision:** `<KEY>-D-<nn>`, numbered within the ADR.
- **Threat:** `<KEY>-T-<nn>`, numbered within the threat model.

Use the IDs everywhere a role refers to a requirement: ADR sections, commit messages, PR descriptions, test names or docstrings, issue bodies, acceptance reports and release notes.

## 2. Artifact status header

Every gated artifact (requirements doc, ADR, test plan, threat model, release plan) starts with this block:

```markdown
> **Status:** Draft | In review | Approved | Superseded
> **Gate:** G1 (or G2, G4…) — approved by <human> on <YYYY-MM-DD>   ← filled only when Approved
> **Trace:** docs/traceability/<feature-slug>.md

| Rev | Date | Change | Affected IDs |
|---|---|---|---|
| 1 | 2026-11-24 | Initial draft | — |
```

- Agents write artifacts as **Draft**, then set them to **In review** when they hand off to a checkpoint.
- **Only the orchestrator sets an artifact to Approved**, and only after the human has explicitly approved it at the gate. An agent never marks its own work Approved.
- Once an artifact is Approved, changing it means adding a revision row. For requirements, it also means a change request (section 4).

## 3. Requirements traceability matrix (RTM)

There is one RTM per feature at `docs/traceability/<feature-slug>.md`. The `requirements-analyst` creates it at the same time as the requirements doc, with one row per AC and NFR. **Each role fills only its own column.** Nobody edits another role's column; if it looks wrong, they route a finding to its owner.

```markdown
# Traceability — <feature name> (<KEY>)

| ID | Story | Design (architect) | Test case(s) (test-engineer) | Implementation (developer) | Verification result (test-engineer) | Defects / findings | State |
|---|---|---|---|---|---|---|---|
| PWD-AC-01 | PWD-US-01 | PWD-D-02 | tests/auth/test_reset.py::test_token_expires | #41 (src/auth/reset.py) | Met — acceptance report 2026-11-30 | — | Verified |
| PWD-AC-02 | PWD-US-01 | PWD-D-02 | tests/auth/test_reset.py::test_reuse_rejected | #41 | Not met | #44 (from:qa) | Open |
```

The *Defects / findings* column is shared. Roles with edit rights (developer, test-engineer) add the issues they file. Read-only roles (code-reviewer, security-auditor) list their issues with AC IDs in their handoff, and the orchestrator records them, which is bookkeeping, not specialized work. The orchestrator also writes any human waivers into the row.

**State** values: `Specified` → `Designed` → `Test-designed` → `Implemented` → `Verified` (or `Open` while a defect is outstanding). A row becomes `Stale` when an approved change request touches it, and downstream columns must then be re-checked.

Gate checks read the RTM directly. For example, G4 cannot pass while any in-scope row is not `Verified` unless the human explicitly waives it, and the waiver is recorded in the row.

## 4. Change requests (after G1)

Once requirements are Approved, nobody edits them quietly, including the human's own mid-sprint ideas. The flow is:

1. Whoever spots the need (human, any agent) raises it. Agents describe it in their handoff. The orchestrator or the `requirements-analyst` files an issue labeled `change-request` (see `GITHUB_CONVENTIONS.md`).
2. The `requirements-analyst` drafts the amendment: new or changed AC IDs, a revision-log row, and the list of affected IDs.
3. The `project-manager` assesses impact on the active sprint (fits / displaces something / goes to backlog) and states it.
4. The human approves or rejects at a mini-G1.
5. If approved: the analyst marks affected RTM rows `Stale`, and the orchestrator re-runs only the downstream steps those rows need (design, test design, implementation, verification).

Clarifications that don't change behavior (an agent consulting the analyst about an existing AC) are **not** change requests. The analyst answers against the approved doc. If answering would require changing an AC, it says so, and that becomes a change request.

## 5. Feedback routing

Problems flow back to **whoever owns the input that caused them**, never to whoever happens to be nearby. Reviewers and auditors describe problems and never fix them.

| Found problem | Routed to | Mechanism |
|---|---|---|
| Code defect, failing test, review Blocker | `developer` | Handoff finding (Blockers) or issue `from:qa` / `from:review` |
| Code diverges from design, or design gap | `architect` | Peer consult; if the design must change, new ADR revision at G2 |
| Ambiguous / missing / contradictory requirement | `requirements-analyst` | Peer consult; if behavior changes → change request |
| Security finding | `developer` (fix) + `architect` if it's a trust-model flaw | Issue `from:security` (private advisory for Critical/High if warranted) |
| Pipeline, deploy or rollback problem | `devops-engineer` | Issue `from:ops` |
| Production incident or post-release defect | `project-manager` → backlog, triaged by severity | Issue `from:ops` + `bug`; urgent ones become a hotfix story |
| Docs don't match behavior | `developer` (if bug) or `architect` (if intent unclear) | Tech-writer consult |
| Scope or priority conflict | Human, via `project-manager` | Checkpoint report |

## 6. Handoff block

Every agent ends its return with this block, so the orchestrator can run the gate check and route findings without re-reading everything:

```markdown
## Handoff
- **Artifacts:** <paths, status of each (Draft / In review)>
- **Trace:** <AC/NFR IDs this run covered; RTM column updated? yes/no>
- **Gate readiness:** <which gate this feeds, which exit criteria are met / not met>
- **Findings routed:** <each finding → owning role, issue URL if filed>
- **Disputes:** <each disagreement raised or answered in this run, in the §7 position format, or "none">
- **Open questions for the human:** <or "none">
- **Suggested next:** <agent + what it should do>
```

## 7. Disagreements

Agents can't hold a conversation. A consult is a single call and nobody shares memory. So a disagreement goes through a short structured exchange and ends with **you**, the human, choosing from a written brief. The aim is to narrow the disagreement into a clear choice, not to let one agent out-argue another.

### 7.1 Decision rights

Every disputed point has an **owner**: the role whose artifact the decision lives in. The owner proposes the agreement. The challenger accepts it or records a dissent. The human decides whenever the parties don't converge, and always has the last word.

| Disagreement about | Owner (proposes) | Lives in |
|---|---|---|
| What a requirement means, whether an AC is right | `requirements-analyst` | requirements doc / change request |
| How something is designed, whether code may diverge from the ADR | `architect` | ADR revision |
| Whether testing is adequate, whether an AC is met, bug vs. intended behavior | `test-engineer` | test plan / acceptance report |
| Security severity, required mitigation | `security-auditor` | threat model / security finding |
| How code is written within the design (structure, naming, local approach) | `developer` | the PR |
| Whether a review finding is valid and how severe it is | `code-reviewer` | the review |
| Release risk, rollback adequacy | `devops-engineer` | release plan |
| Scope, priority, sprint impact | `project-manager` | backlog / sprint milestone |
| Whether docs describe intended behavior | Owner of the intent (`architect` or `developer`) | docs + ADR |

**No agent can overrule these:** an unresolved review **Blocker**, a **Critical/High** security finding, or an AC marked **not met**. An agent can dispute any of them, but only the human can accept the risk, and that acceptance is recorded (§7.4). Until the human decides, the gate stays closed.

### 7.2 Raising a disagreement: one round, then escalate

1. **Challenger states a position**, in this format, in its handoff and, if it can, in a consult to the owner:
   ```markdown
   **Position (<role>) on <topic>:** <what I think should happen, one sentence>
   - Against: <the artifact/decision/finding I disagree with, with ID>
   - Evidence: <file:line, test output, AC/threat/decision IDs, measurements; not opinion>
   - Cost if I'm wrong / cost if they're wrong: <one line each>
   - Would change my mind: <what evidence or condition would settle it for me>
   ```
2. **Owner responds once.** It either:
   - **concedes**: revises its own artifact (new revision row, affected IDs listed), and the dispute is resolved; or
   - **holds**: gives its own position in the same format, plus a **proposed agreement**, the option it recommends to the human. That may be its original position, the challenger's, or a compromise (e.g. "async only for this endpoint", "accept the risk for this sprint and file a follow-up").
3. **No further rebuttals.** If the owner held, the dispute escalates to the human. Agents don't go back and forth. Every extra round costs a full invocation and adds nothing a human decision wouldn't settle faster.

Agents that can consult the owner directly (developer → architect, test-engineer → requirements-analyst, etc.) run step 2 inside their own run. Where there's no consult route (code-reviewer vs. developer, developer vs. test-engineer), the challenger only writes its position in its handoff, and the orchestrator runs the owner's one response (see `CLAUDE.md`).

### 7.3 Decision brief

When a dispute escalates, the orchestrator writes a decision brief from the two positions and puts it in front of the human. The orchestrator **does not pick a side**. Its job is to make the choice clear: options in neutral wording, evidence attributed to whoever supplied it, and the owner's proposed agreement labeled as the owner's.

```markdown
## Decision needed: DR-<KEY>-nn, <short title>
Raised at: <gate / step>   Parties: <challenger> vs <owner (owner)>   Affects: <AC/T/D IDs>

Question: <the one thing to decide, phrased as a choice>
Option A (<role>): <position>. Gains: <…>. Costs: <…>.
Option B (<role>): <position>. Gains: <…>. Costs: <…>.
Evidence: <the strongest evidence each side cited, attributed>
Proposed agreement (owner, <role>): <option or compromise>. Challenger: accepts / dissents: "<one line>"
Downstream impact: <artifacts to revise, tests to add/drop, sprint impact (from PM if material)>
Gate effect: <which gate stays closed until this is decided>
Your call: A / B / proposed agreement / something else
```

Number decisions `DR-<KEY>-nn` per feature, or `DR-nn` for project-wide ones.

### 7.4 Recording the decision

Once the human decides:
1. The orchestrator appends an entry to `docs/decisions.md`: ID, date, question, decision, who decided, the dissent if there was one, and links.
2. The owner applies the decision to its artifact (ADR revision, change request, test plan revision, finding downgraded or closed, release plan update), referencing the DR ID. A decision that changes approved requirements still goes through a change request (§4); the decision record is the approval.
3. A risk the human accepted (an overruled Blocker, Critical/High finding or failed AC) is also recorded in the affected RTM rows as a waiver referencing the DR ID, with a follow-up issue filed.
4. **Disagree and commit.** The parties work to the decision from then on. Nobody reopens it in a later run unless they have **new evidence**, the condition in their "would change my mind" line, in which case it's a new dispute referencing the old DR.

Recurring disputes of the same kind are input for the retro. They usually mean a convention, template or decision right is unclear.
