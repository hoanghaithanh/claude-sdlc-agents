# SDLC Agent Team — Claude Code Subagent Template

A set of Claude Code subagents, one per SDLC role, meant to be dropped into any project's `.claude/agents/` directory. Claude Code will auto-invoke the right one based on the task, or you can call one by name.

## Roles included

| Agent | SDLC phase | Model / effort | Can write code? |
|---|---|---|---|
| `project-manager` | Planning / tracking / close-out | sonnet / medium | No — owns milestones, backlog, retros |
| `requirements-analyst` | Requirements | sonnet / high | No — writes specs only |
| `architect` | Design | **opus / high** | No — writes design docs only |
| `developer` | Implementation | sonnet / medium | Yes |
| `code-reviewer` | Review | sonnet / medium | No — read-only (files issues only) |
| `test-engineer` | Test design, testing, acceptance validation | sonnet / medium | Tests + test plans only |
| `security-auditor` | Threat model + security review | **opus / high** | No — read-only toward code (writes threat models only) |
| `devops-engineer` | Release/deploy + post-release checks | sonnet / medium | Infra/CI config + release plans only |
| `tech-writer` | Documentation | sonnet / medium | Docs only |

(Tool lists per agent are in each agent's frontmatter; reviewers and auditors are read-only toward code.)

Tool access is deliberately narrow per role — reviewers and auditors are read-only by design, so they can't quietly "fix" what they were asked to only assess.

## Model tiering (cost vs. depth)

Agents are split into two tiers so you're not paying top-tier prices for mechanical work:

- **Deep-thinking tier — `model: opus`, `effort: high`**: `architect`, `security-auditor`. These make expensive-to-reverse calls — choosing an architecture, reasoning about trust boundaries — so they get the most capable model and think harder.
- **Reasoning-heavy but cheaper — `model: sonnet`, `effort: high`**: `requirements-analyst`. Requirements work benefits from deep reasoning (high effort) but not necessarily the opus price, so it runs sonnet at high effort. Promote to opus with a one-line frontmatter edit if requirements quality becomes a pain point.
- **Execution tier — `model: sonnet`, `effort: medium`**: everyone else (`developer`, `code-reviewer`, `test-engineer`, `devops-engineer`, `tech-writer`, `project-manager`). Bounded, well-specified work where sonnet at medium effort matches opus quality at a fraction of the cost and latency.

Both `model:` and `effort:` are per-agent frontmatter fields Claude Code reads at invocation time. `effort` controls how much the model reasons before acting (`low`/`medium`/`high`/`max`), independent of which model runs.

### Things to know

- **Model names may drift.** These files use the tier aliases `opus` and `sonnet` rather than pinned IDs (like `claude-opus-4-8`) on purpose — aliases keep the files portable and always resolve to the current model in that tier. If you need a specific pinned version for reproducibility, replace the alias with the full model ID in that agent's frontmatter.
- **Requires a plan/tier with Opus access.** If your Claude Code plan doesn't include Opus, those three agents will fall back to whatever's available (often Sonnet) — the split then does nothing, but nothing breaks. On metered billing, the opus agents are the expensive ones; watch them first.
- **`effort` is a relatively new field.** Per-subagent `effort` in frontmatter is supported, but if you're on an older Claude Code build that only reads session-level effort, the per-agent value is simply ignored and the model still runs — you just lose the fine-grained control. The `model:` split still works regardless.
- **Two judgment calls worth revisiting for your project:**
  - `security-auditor` is in the opus tier because trust-boundary reasoning is design-level, even though it isn't literally "architecture." `requirements-analyst` was moved to sonnet/high as a cost trade-off. If you'd rather scope the opus tier strictly to `architect`, drop the auditor too; if requirements quality matters more than cost, promote the analyst back to opus.
  - `code-reviewer` is kept on sonnet, since pattern-and-style review is where sonnet performs well (this mirrors Anthropic's own built-in agents). If your reviews need deeper architectural judgment, promote it to opus — at higher cost per review.
- **Global override exists.** `CLAUDE_CODE_SUBAGENT_MODEL` overrides every agent's model at once (useful to force everything to sonnet during development to save cost). Per-agent frontmatter is what gives you this tiering; the env var is the blunt instrument.

## How the roles interact: artifacts, gates, trace

The agents are wired together the way roles are in a gated SDLC. They don't share memory, so everything passes through artifacts, human gates, and a traceability matrix. The contracts live in **`.claude/SDLC_CONVENTIONS.md`**; the gates live in **`CLAUDE.md`**.

| Gate | Approves | Main artifacts |
|---|---|---|
| **G1** Requirements baseline | Testable ACs with stable IDs | `docs/requirements/`, `docs/traceability/` (RTM created) |
| **G2** Design approved | Design, test plan, and threat model together (V-model shift-left) | `docs/architecture/`, `docs/qa/test-plans/`, `docs/security/` |
| **G3** Code complete | Tests pass, no Blockers or Critical/High findings | PR with AC IDs, RTM implementation column |
| **G4** Acceptance | Every AC verified or waived; **human sign-off** | `docs/qa/acceptance/` |
| **G5** Go / no-go | Release plan with rollback; human as change board | `docs/releases/`, release notes |
| **G6** Close-out | Post-release checks; feedback into backlog | `from:ops` issues, `docs/backlog.md`, retro |

- **Traceability.** Every acceptance criterion (`PWD-AC-03`) is tracked in `docs/traceability/<feature>.md` from design through planned tests, code, verification and defects. Each role fills only its own column.
- **Change control.** After G1, a requirement changes only through a `change-request` issue: the analyst amends, the PM assesses sprint impact, the human approves, and affected trace rows go `Stale` and get re-checked.
- **Feedback routing.** Findings go to the role that owns the faulty input (defect → developer, design gap → architect, requirement gap → analyst, production issue → PM → backlog). Production feedback re-entering the backlog is what closes the cycle.
- **Handoff block.** Every agent ends with a standard `## Handoff` section (artifacts, trace, gate readiness, routed findings, next step), so the main session can run the gate check.
- **Disagreements.** Each kind of decision has an owner (design → architect, severity → security-auditor, …). The challenger states an evidence-based position, and the owner responds once: it either concedes or holds with a proposed agreement. If they still disagree, the main session writes a neutral decision brief and **you** decide. The decision is logged in `docs/decisions.md`, and the parties work to it ("disagree and commit"). Agents can't overrule a Blocker, a Critical/High security finding or a failed AC between themselves; only you can accept that risk.
- **Two lanes.** Small fixes take a **fast lane** (developer → test → review → one combined approval), so the gates don't tax trivial work.

## Inter-agent communication (peer consultation)

Within a single run, agents can also consult each other directly:

Agents can consult each other when they hit a genuine ambiguity, instead of silently guessing — the way the real roles would:

```
                          requirements-analyst
                            ↑              ↑
              consulted by architect   consulted by test-engineer
                            ↑
   architect  ←── consulted by developer, code-reviewer,
                  security-auditor, devops-engineer, tech-writer
```

- **architect → requirements-analyst**: when a requirement is ambiguous or missing detail that would change the design
- **developer → architect**: when a design doc doesn't specify how to handle something that changes the implementation
- **test-engineer → requirements-analyst**: when expected behavior for an edge case isn't defined
- **code-reviewer / security-auditor → architect**: when code deviates from a design doc and it's unclear whether that's deliberate
- **devops-engineer → architect**: when the deployment topology depends on an architectural decision not yet recorded
- **tech-writer → developer / architect**: when code behavior is ambiguous and it's unclear whether it's intended (a feature to document) or a bug

Each of these is scoped via `Agent(agent-name)` in the calling agent's `tools:` frontmatter, so e.g. the developer can only escalate to the architect, not spawn arbitrary agents.

### Requirements and caveats — read before relying on this

This depends on **nested subagent spawning**, a capability Claude Code shipped on **June 10, 2026 (v2.1.172)**. Before that version, subagents could not call other subagents at all — if you're on an older version, update first (`npm i -g @anthropic-ai/claude-code`).

A few things to know since this is a very new capability:

- **It's experimental and not fully reliable.** Models don't always nest on their own even when the prompt tells them to — you may sometimes need to explicitly say "ask the architect agent about X" yourself rather than trusting it to happen automatically.
- **Nesting is capped at 5 levels** by Claude Code itself, so there's no risk of infinite recursion, but each level adds latency and token cost — a developer asking an architect asking a requirements-analyst is 3 full agent invocations for one clarification.
- **No shared memory between agents.** When one agent consults another, it has to restate all relevant context in the delegation prompt — the target agent has never seen the conversation. Each agent's instructions above are written to do this explicitly, but if you notice a consultation coming back generic or off-target, the calling agent probably under-specified the question.
- **This escalates cost, not just capability.** More nesting means more Claude Code invocations per task. If you're on a budget or these clarifying round-trips aren't adding value, drop the `Agent(...)` entries from the tools list and the agents fall back to stating assumptions explicitly instead — which is often good enough.

## Install

Copy the whole `.claude/agents/` folder into your project root:

```bash
cp -r .claude/ /path/to/your/project/
```

Or install a single agent globally (available across all your projects):

```bash
cp .claude/agents/code-reviewer.md ~/.claude/agents/
```

Verify they loaded with `/agents` inside a Claude Code session.

## GitHub integration

Agents interact with GitHub through the `gh` CLI (run via Bash), split by role the way a real team would be:

- **project-manager owns milestones** — the high-level, release-level view. It creates, tracks, and closes GitHub milestones and assigns issues to them. It does *not* file day-to-day bugs.
- **Cross-functional agents log day-to-day issues** — developer, test-engineer, code-reviewer, security-auditor, and requirements-analyst each file GitHub issues within their own concern (a QA bug, a review finding, a security finding, a follow-up task), labeled by role, and attach them to a milestone the PM defined. They never create or close milestones.

Shared issue conventions (labels, body format, dedup rules) live in `.claude/GITHUB_CONVENTIONS.md`, which the agents reference.

### Prerequisites — do this before the GitHub features work

1. **Install and authenticate `gh`** in the environment where Claude Code runs:
   ```bash
   gh auth login          # interactive, one-time
   gh auth status         # verify
   ```
   The token needs `repo` scope (issues + milestones live under it).
2. **Milestones use the REST API**, not a native `gh` command — `gh` has no `gh milestone`. The project-manager agent already has the correct `gh api` syntax baked in, so you don't need the third-party milestone extension.
3. **Agents act as whoever `gh` is authenticated as.** Every issue/milestone will be attributed to that account. Use a dedicated bot account if you don't want the activity mixed with a human's.

### Safety notes

- These agents can **write to your real GitHub repo** (create issues, milestones, comments). Test against a scratch repo first.
- The agents are instructed to search before filing to avoid duplicates and never to run destructive operations (deleting issues/milestones) without stating what will be removed — but review the first few runs, since automated issue-filing can get noisy fast.
- The security-auditor is instructed to flag rather than auto-post sensitive findings publicly, but confirm it's behaving that way before trusting it on a real repo.

## Orchestration & workflow

The pipeline order and the definition-of-done live in **`CLAUDE.md`**, which the main Claude Code session reads automatically. That's what turns the individual agents into a process.

**Human-checkpointed, not auto-piloted.** The main session runs the pipeline **one stage at a time**: after each agent finishes, it stops, reports back what was produced and what's next, and waits for your go-ahead before invoking the next agent. It won't chain the whole pipeline unattended — you can review each artifact, answer open questions, or skip steps at every handoff. (An agent consulting a peer *within its own run* doesn't checkpoint — that's internal to that agent.)

Default full-lane pipeline (full detail, exit criteria, and the greenfield variant are in `CLAUDE.md`):

1. **project-manager**: sprint/milestone
2. **requirements-analyst**: requirements, IDs, RTM → **G1**
3. **architect**: design mapped to AC IDs
4. **test-engineer (test design)**: test plan from the ACs, before code exists
5. **security-auditor (threat model)**: if sensitive → **G2**
6. **developer**: implement, with AC IDs in commits/PR
7. **test-engineer**: implement and run the planned tests
8. **code-reviewer**: review the diff, including a trace check
9. **security-auditor (code audit)**: if sensitive, verify the mitigations → **G3**
10. **test-engineer (acceptance validation)**: then **you give final sign-off** → **G4**
11. **tech-writer**: docs + release notes
12. **devops-engineer (release readiness)**: release plan + rollback → **G5**
13. **devops-engineer (post-release check)**: production issues filed `from:ops`
14. **project-manager**: close-out, feedback into backlog, retro → **G6**

You don't have to invoke every agent for every task. A one-line fix goes through the fast lane (developer → test-engineer → code-reviewer → one approval). You can also invoke any agent directly by name at any time, bypassing the pipeline.

### Starting a project from scratch

`CLAUDE.md` has a dedicated greenfield sequence. The key differences from the feature loop: get one **thin end-to-end slice** through build→deploy→test→review before building features (surfaces architecture/integration problems while they're cheap), scaffold **CI/deploy early** (Phase 2, not at the end), and **don't invoke the reviewer, security-auditor, or tech-writer on day one** — they have no code to work on yet.

## Customizing

- Edit any agent's frontmatter `tools:` list to loosen/tighten permissions.
- Edit `model:` per agent if you want cheaper/faster models on lower-stakes roles (e.g. `haiku` for tech-writer) and stronger models on higher-stakes ones (e.g. `opus` for architect or security-auditor).
- Add project-specific conventions (tech stack, folder layout, coding standards) directly into each agent's body — the more specific the instructions, the better the output.
- Keep each agent's job narrow. If you find yourself wanting one agent to do two very different things, split it instead of expanding it — that's what keeps delegation decisions clean.
