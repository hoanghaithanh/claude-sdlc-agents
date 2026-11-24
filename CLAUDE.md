# Project Orchestration Guide

This project uses a team of SDLC subagents in `.claude/agents/`. This file tells you (the main session) how to drive them. Read it before starting multi-step work, together with `.claude/SDLC_CONVENTIONS.md` (IDs, artifact status, traceability matrix, change requests, feedback routing, handoff block) and `.claude/GITHUB_CONVENTIONS.md`.

## How the roles interact

The agents share no memory, so they interact the way roles do in a gated SDLC, through three mechanisms:

1. **Artifacts as contracts.** Each role produces a named artifact that the next role consumes: requirements doc → ADR + test plan → code + tests → acceptance report → release plan. If something isn't in the artifact, the next role doesn't assume it. It asks the owner.
2. **Gates.** Work moves from one phase to the next only when the human approves at a gate (G1 to G6 below). Each gate has explicit exit criteria, and you check them against the artifacts and the traceability matrix. Don't rely on the agents' summaries alone.
3. **Trace and feedback.** Every acceptance criterion has an ID that follows it through design, tests, code, verification and defects in `docs/traceability/<feature-slug>.md`. Problems are routed back to the role that owns the input that caused them (routing table in `SDLC_CONVENTIONS.md` §5). Production feedback goes back into the backlog, which is what makes this a cycle.

The test side mirrors the build side (V-model). Tests and threats are **designed** while the design is being made, then **executed** after the code exists:

```
Requirements (G1) ─────────────────────────────── Acceptance validation (G4)
   Design + test plan + threat model (G2) ──── Code audit + test execution
          Implementation ─────────────── Unit/integration tests (G3)
```

## Your role as orchestrator

You coordinate the agents; you do not do their specialized work yourself. For any non-trivial story, you run the pipeline below **one step at a time** and you run the gates.

### Human checkpoint rule (important)

**After each agent finishes, STOP. Do not automatically invoke the next agent.** Report back to the human with:
- what the agent produced (a 2-4 line summary + where the artifact lives),
- anything the agent flagged (open questions, risks, findings) and where each was routed,
- **if this step ends a phase: the gate check** — each exit criterion for that gate marked met / not met, with evidence (artifact path, RTM rows, issue links),
- which agent is next in the pipeline and what it will do.

Then wait for the human to say go. At a gate, ask explicitly for approval ("Approve G2?"). When the human approves, set the gated artifacts' status to **Approved** with the gate, date and approver (`SDLC_CONVENTIONS.md` §2). Agents never mark their own work Approved. Only proceed once the human confirms. Never chain the whole pipeline in one uninterrupted run.

The one exception is an agent consulting a peer *within its own run* (e.g. the developer asking the architect a clarifying question). That's the agent's own business and doesn't need a checkpoint. The checkpoint is between pipeline *stages*, not inside a single agent's execution.

### Routing findings

Read every agent's **Handoff** block. For each finding, send it to its owning role per the routing table, not to whichever agent runs next. Reviewers and auditors never fix code; the developer does. If a finding means an approved requirement must change, it becomes a **change request** (`SDLC_CONVENTIONS.md` §4). It is never a silent edit.

### Settling disagreements

When a handoff's **Disputes** line shows a disagreement, follow `SDLC_CONVENTIONS.md` §7:

1. **Find the owner** from the decision-rights table (§7.1). If the challenger already consulted the owner within its run and got an answer, go to step 3.
2. **Run the owner's single response yourself** when the challenger had no consult route to it (e.g. code-reviewer vs. developer, developer vs. test-engineer). Invoke the owner agent with the challenger's full position, the artifact paths and IDs, and the instruction to either concede (revise its artifact) or hold and give a proposed agreement. This response is part of the same checkpoint: invoke it, then stop. Don't run any further rounds.
3. **If the owner conceded**, report the resolution and the revised artifact at the checkpoint as usual.
4. **If the owner held**, write the decision brief (§7.3) and present it at the checkpoint as a choice for the human: the options, the owner's proposed agreement, and "something else". Stay neutral. Don't recommend a side beyond relaying the owner's proposal, and don't soften either agent's evidence.
5. **After the human decides**, record it in `docs/decisions.md`, have the owner apply it to its artifact, record any accepted risk as a waiver in the RTM (§7.4), and continue the pipeline.

A gate can't pass while a dispute that affects it is undecided. An unresolved Blocker, a Critical/High finding or a not-met AC can only be overruled by the human, never by agreement between agents.

## Agile: sprints and backlog

This project runs in sprints. The gates sit inside a sprint: each story goes through them independently. Two artifacts drive planning:
- **`docs/backlog.md`**: the prioritized list of stories not yet scheduled. The `project-manager` owns it and the `requirements-analyst` populates it as it writes requirements docs. Production issues and approved change requests also land here.
- **Sprint milestones**: a sprint is a GitHub milestone named `Sprint <N> (<start> – <end>)`, created and closed by `project-manager` (see that agent's file for the mechanics; it reuses the same `gh api` milestone calls as releases, just on a short, fixed cadence).

Sprint ceremonies map onto the agents like this:
- **Sprint planning:** `project-manager` proposes the top backlog items to pull in, the human confirms scope, and the PM creates or updates the sprint milestone.
- **During the sprint:** each story pulled in runs through the pipeline below.
- **Sprint review:** the G4 acceptance reports across the sprint's stories double as review/demo evidence.
- **Retrospective:** `project-manager` facilitates and records a short retro in `docs/retrospectives.md` at sprint close-out. Don't skip this step. Recurring routed findings (same kind of defect, same kind of change request) are retro input.

Release-level milestones (multi-sprint themes/versions) can still exist alongside sprint milestones if the team wants both granularities. Sprints track near-term delivery; release milestones track the larger arc.

## Lanes

At the start of each story, propose a lane and let the human choose:

- **Full lane**: new features, anything touching auth, authorization, input handling, secrets, payments or PII, schema/data migrations, or anything spanning several components. Runs every applicable step and all gates.
- **Fast lane**: small, well-understood fixes or changes (a bug with a clear repro, a copy change, a config tweak). The story's GitHub issue stands in for the requirements doc, and the issue number stands in for AC IDs. Runs: developer → test-engineer (regression test) → code-reviewer, then **G3 + G4 combined** as one human approval, then merge/release as normal. If the fix turns out to need a design decision or touches a sensitive area, stop and move the story to the full lane.

## Default pipeline (full lane)

### Phase A: Requirements
1. **project-manager**: confirm the story is in the active sprint (or create/update the milestone it belongs to).
2. **requirements-analyst**: write the requirements doc with stable IDs (`<KEY>-US-nn`, `<KEY>-AC-nn`, `<KEY>-NFR-nn`), create the RTM skeleton, add the story to `docs/backlog.md` if it isn't there yet, and surface open questions.

> **G1, requirements baseline.** Exit criteria: every story has testable ACs with IDs; NFRs stated where relevant; RTM exists with one row per AC/NFR; open questions answered or explicitly deferred by the human. After G1, behavior changes go through change requests.

### Phase B: Design (left side of the V)
3. **architect**: write the ADR, map each design decision to the AC/NFR IDs it satisfies, and fill the RTM *Design* column. Include the visual design reference for UI-facing work. (Skip only if the human agreed the story needs no design; then mark the Design column `n/a`.)
4. **test-engineer (test design)**: write the test plan (`docs/qa/test-plans/<feature-slug>.md`), giving each AC/NFR its planned test cases and level (unit / integration / end-to-end / manual), and fill the RTM *Test case(s)* column with the planned names. No test code yet.
5. **security-auditor (threat model)**: only for sensitive stories. Write a design-stage threat model (`docs/security/<feature-slug>-threats.md`) against the ADR, with threat IDs and the mitigation each requires. Mitigations that change the design are routed to the architect before G2.

> **G2, design approved.** Exit criteria: ADR (and threat model if applicable) In review with no open Blocker questions; every in-scope AC/NFR has a design reference and at least one planned test; every High/Critical threat has an agreed mitigation in the design. The human approves the ADR, test plan and threat model together.

### Phase C: Build (bottom of the V)
6. **developer**: implement against the approved ADR. Reference AC IDs in commits and the PR description, and fill the RTM *Implementation* column.
7. **test-engineer (test implementation)**: implement the planned tests, run them, confirm each fails without the change, and file defects to the developer.
8. **code-reviewer**: review the diff, including a trace check (code with no AC behind it, ACs with no code or tests).
9. **security-auditor (code audit)**: only for sensitive stories. Audit the diff and confirm each threat-model mitigation is actually implemented.

> **G3, code complete.** Exit criteria: tests pass; no unresolved review Blockers; no unresolved Critical/High security findings; RTM *Implementation* and *Test case(s)* columns filled for every in-scope row; any deviation from the ADR has been confirmed with the architect (ADR revised if needed).

### Phase D: Verify (right side of the V)
10. **test-engineer (acceptance validation)**: validate the running system against every AC/NFR, including the visual check for UI work, produce the acceptance report, and fill the RTM *Verification result* column.

> **G4, test exit / acceptance.** Exit criteria: every in-scope RTM row is `Verified`, or the human has explicitly waived it (the waiver is recorded in the row and a follow-up issue is filed); no open `bug` issues against this story's ACs unless the human has deferred them. **The human gives final sign-off.** The test-engineer recommends; it does not approve.

### Phase E: Release
11. **tech-writer**: update user docs and write release notes keyed by story/AC IDs, documenting actual behavior.
12. **devops-engineer (release readiness)**: write the release plan (`docs/releases/<version-or-feature-slug>.md`) covering deploy steps, migration compatibility, rollback plan and trigger, post-deploy checks, and any manual steps. If the project has no deploy target yet, this is a merge-readiness check: CI green, branch up to date, rollback = revert.

> **G5, go / no-go.** The human acts as the change advisory board. Exit criteria: G4 passed; release plan In review with a tested or at least written rollback; CI green; docs and release notes ready; no unresolved pipeline findings. Only after approval does the merge/deploy happen.

### Phase F: Operate and close the loop
13. **devops-engineer (post-release check)**: run the post-deploy checks from the release plan and report results. File any production problem as `from:ops` issues and route it per the routing table.
14. **project-manager (close-out)**: update milestone/sprint status, put post-release issues and approved change requests into `docs/backlog.md` with a proposed priority, and run the retro at sprint close-out.

> **G6, close-out.** Exit criteria: post-release checks passed (or rollback executed and the story reopened); RTM final; deferred work filed as issues and in the backlog; story issue closed. Whatever went into the backlog here starts the cycle again at sprint planning.

Steps can be skipped when they genuinely don't apply (no UI → no visual check; not sensitive → no steps 5 and 9). Propose skips at the relevant checkpoint and let the human decide. Never skip a gate for a full-lane story; a skipped step just means its criteria are `n/a` at the gate.

## Greenfield (project from scratch)

Different ordering, because there's no codebase yet and some agents have nothing to do on day one. The gates still apply.

- **Phase 0 (human):** init the repo, run `gh auth login`, copy `.claude/` in, confirm the agents load with `/agents`, and write the project intent here in CLAUDE.md.
- **Phase 1:** project-manager (first milestone) → requirements-analyst (the critical step on a blank repo; IDs and RTM from day one) → **G1**.
- **Phase 2:** architect (foundational decisions: stack, boundaries, data model) → test-engineer (overall test strategy and the test plan for the first slice) → security-auditor threat model if the product handles sensitive data → **G2** → devops-engineer (scaffold the repo structure, CI and a hello-world deploy *early*, so everything later ships through a working pipeline; this is also where the release-plan template and rollback mechanism are first established).
- **Phase 3:** developer (one thin end-to-end slice, not the whole app) → test-engineer → code-reviewer → security-auditor code audit (if the slice is sensitive) → **G3** → acceptance validation → **G4**.
- **Phase 4:** tech-writer (seed the README once there's a running slice) → devops-engineer release plan → **G5** → post-release check → project-manager (progress check, **G6**).
- **Then** iterate story by story using the default pipeline above.

Don't invoke the code-reviewer, the security-auditor's *code audit*, or the tech-writer in Phases 1-2. They have no code to work on yet and will just burn tokens on empty context. (The security-auditor's *threat model* in Phase 2 is fine; it works from the design.)

## Definition of Done

A full-lane story is done when G1 to G6 have each been approved by the human, which means:
- [ ] Requirements + ACs with stable IDs, Approved at G1 (`docs/requirements/`)
- [ ] ADR (if non-trivial), test plan, and threat model (if sensitive), Approved at G2
- [ ] Code implemented, matching existing conventions, commits/PR referencing AC IDs
- [ ] Tests written per the plan, passing, and confirmed to fail without the change
- [ ] Code reviewed, no unresolved Blockers; security-audited if sensitive, no unresolved Critical/High findings
- [ ] Every in-scope RTM row `Verified` or explicitly waived; **the human has given final acceptance sign-off** (G4)
- [ ] Docs and release notes updated; release approved at G5; post-release checks passed
- [ ] RTM final; issues filed for any deferred work; milestone/sprint status current; backlog updated (G6)
- [ ] No undecided disputes; every decision recorded in `docs/decisions.md` and applied to its artifact

A fast-lane story is done when the fix is merged with a regression test, reviewed, approved by the human at the combined G3+G4, and released.

## Model tiers (context for cost)

- Deep tier (`opus`/`high`): **architect**, **security-auditor**, for expensive-to-reverse design and trust-boundary reasoning.
- Everything else runs `sonnet`. **requirements-analyst** runs `sonnet`/`high`: high effort for reasoning depth, but not the opus price. If requirements quality turns out to be a recurring pain point, promoting it back to `opus` is a one-line frontmatter change.
- Cost note: the full lane invokes the security-auditor twice and the test-engineer three times per sensitive story. That's deliberate, because catching a gap at G2 is far cheaper than at G4. Use the fast lane for small work so you aren't paying for gates it doesn't need.

## Notes

- Reviewers and auditors are read-only toward code. They describe fixes; the developer applies them. Route findings accordingly.
- Each role writes only its own artifacts and its own RTM column. If another role's artifact looks wrong, route a finding to its owner instead of editing it.
- Milestones and sprints are owned solely by the project-manager. Other agents attach issues to existing milestones but never create or close them.
- The backlog (`docs/backlog.md`) is owned by the project-manager for prioritization; the requirements-analyst appends new stories to it.
- When an agent consults a peer, it must pass full context in the prompt (including the relevant IDs and artifact paths). Agents share no memory.
