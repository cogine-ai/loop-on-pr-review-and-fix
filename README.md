# Loop on PR Review and Fix

**Turn PR review feedback into a disciplined verify-fix-validate loop.**

Loop on PR Review and Fix is a portable agent skill for handling GitHub pull request feedback after a PR has been opened. It watches for new review comments, verifies each claim against the current code, fixes only still-valid issues, validates the change, pushes the PR branch, and remembers what was already handled.

It is built for one thing: **close the loop on real PR feedback without blindly applying stale or speculative suggestions.**

## Install

```bash
npx skills add https://github.com/cogine-ai/loop-on-pr-review-and-fix --skill loop-on-pr-review-and-fix
```

Then invoke it from a skill-aware agent:

```text
$loop-on-pr-review-and-fix
```

## Why This Exists

PR review automation usually breaks down in predictable ways:

- it re-handles comments that were already fixed or resolved
- it applies reviewer feedback without checking whether it is still valid
- it treats outdated inline comments as current code defects
- it makes broad fixes for narrow review comments
- it forgets state between polling rounds
- it keeps running after the PR has gone quiet

Loop on PR Review and Fix makes the loop explicit: fetch review state, compare it with durable local state, verify each new item, make the smallest safe fix, validate, push, and record the outcome.

## Default Command

```text
$loop-on-pr-review-and-fix
```

Default Codex behavior:

- creates a heartbeat automation first
- wakes every 10 minutes
- runs exactly one review-check iteration per wakeup
- stores handled review state in `.git/codex-pr-review-loop/`
- fixes only new actionable review feedback
- skips resolved, obsolete, duplicate, not-actionable, or intentionally skipped feedback
- stops after 3 consecutive quiet rounds

The default setup turn does **not** immediately edit code. It creates the loop. Ask for an immediate pass explicitly when you want one.

## Command Examples

Start a Codex heartbeat loop for the active PR:

```text
Use $loop-on-pr-review-and-fix for this PR.
```

Start a loop for a specific PR and repository path:

```text
Use $loop-on-pr-review-and-fix for https://github.com/org/repo/pull/123 in /absolute/path/to/repo.
```

Run one immediate pass instead of creating only the automation:

```text
Use $loop-on-pr-review-and-fix in Iteration Mode for PR 123 in /absolute/path/to/repo. Run exactly one review-check iteration now.
```

Claude Code style explicit loop:

```text
Use $loop-on-pr-review-and-fix for PR 123. Run one iteration, wait 10 minutes if the loop should continue, then repeat until three quiet rounds.
```

## Modes

| Mode | Trigger | Behavior |
| --- | --- | --- |
| Setup Mode | User asks to start, watch, monitor, loop on, or automatically handle PR review feedback | Create a 10-minute heartbeat automation and stop |
| Iteration Mode | Heartbeat wakes, or user explicitly asks for one immediate pass | Fetch review state, handle new actionable feedback once, update state |
| Claude Code Mode | No Codex heartbeat tool is available | Use an explicit agent-controlled loop with 10-minute waits |

If unsure, the skill prefers Setup Mode. That keeps code changes intentional instead of turning every "watch this PR" request into an immediate edit.

## Design Mapping

| PR review loop problem | Skill mechanism |
| --- | --- |
| Repeated comments across polling rounds | Durable local state file |
| Stale or outdated inline comments | Verify against current code before fixing |
| Resolved review threads | Skip resolved items |
| Duplicate bot comments | Stable item ids and handled-status tracking |
| Blindly applying suggestions | Treat every comment as a claim to verify |
| Unbounded polling | Quiet-round counter with stop condition |
| Fragile shell loops | Agent performs each inspect-edit-validate-commit cycle |

## Workflow

1. **Resolve context**: identify repo, PR number, PR URL, head branch, base branch, and current head SHA.
2. **Initialize state**: create or update the local state file for this PR.
3. **Fetch review state**: collect PR comments, reviews, review threads, thread resolution status, and outdated markers.
4. **Classify feedback**: exclude handled, resolved, obsolete, duplicate, not-actionable, and intentionally skipped items.
5. **Verify new items**: inspect current code and confirm whether each comment still describes a real issue.
6. **Fix minimally**: edit only what is needed for still-valid actionable feedback.
7. **Validate**: run focused tests first, then broader checks if shared behavior changed.
8. **Commit and push**: stage only relevant files, commit, and push to the PR branch.
9. **Record outcome**: mark every item as fixed, skipped, not-actionable, obsolete, duplicate, or resolved.
10. **Continue or stop**: reset quiet rounds after action; increment quiet rounds when nothing new is actionable; stop after 3 quiet rounds.

## What Counts as Actionable

Actionable feedback must be:

- new to the loop
- still valid against the current code
- tied to a concrete code path or review thread
- fixable without changing product direction
- narrow enough to validate in this PR

The loop skips:

- resolved threads
- outdated comments whose concern no longer applies
- duplicate findings
- status chatter, praise, acknowledgements, or already-answered questions
- speculative advice with no current failure path
- product-level ambiguity that needs human decision
- items explicitly skipped by the user or a prior loop round

## State Model

The helper script stores loop state under the local git directory by default:

```text
.git/codex-pr-review-loop/pr-<number>.json
```

Handled statuses:

| Status | Meaning |
| --- | --- |
| `fixed` | A relevant fix was committed and pushed |
| `skipped` | Intentionally not fixed, with a reason |
| `not-actionable` | Comment does not require a code change |
| `obsolete` | Feedback no longer applies to current code |
| `duplicate` | Same issue already handled elsewhere |
| `resolved` | Review thread is already resolved |

## What's Inside

```text
loop-on-pr-review-and-fix/
├── SKILL.md
├── agents/
│   └── openai.yaml
└── scripts/
    └── review_loop_state.py
```

Key files:

- `SKILL.md`: agent entrypoint, mode selection, guardrails, and loop workflow
- `agents/openai.yaml`: skill list metadata for OpenAI-compatible skill UIs
- `scripts/review_loop_state.py`: deterministic local state helper

## Optional Script Usage

Most users should invoke the skill from an agent. The helper script exists so agents can manage loop state reliably instead of rewriting state bookkeeping.

Initialize or update PR state:

```bash
python3 scripts/review_loop_state.py init \
  --pr 123 \
  --repo org/repo \
  --pr-url https://github.com/org/repo/pull/123 \
  --head-sha HEAD_SHA
```

Show state summary:

```bash
python3 scripts/review_loop_state.py summary --pr 123
```

Print item ids that are not already handled:

```bash
python3 scripts/review_loop_state.py unseen --pr 123 comment:abc thread:def
```

Mark an item handled:

```bash
python3 scripts/review_loop_state.py mark \
  --pr 123 \
  --item-id comment:abc \
  --status fixed \
  --note "Adjusted validation path" \
  --head-sha NEW_HEAD_SHA
```

Track quiet rounds:

```bash
python3 scripts/review_loop_state.py quiet --pr 123 --increment --stop-after 3
python3 scripts/review_loop_state.py quiet --pr 123 --reset
```

To store state somewhere else:

```bash
PR_REVIEW_LOOP_STATE_DIR=/tmp/pr-review-loop \
  python3 scripts/review_loop_state.py summary --pr 123
```

## Relationship to Local Ultra Review

This skill is a companion to [Local Ultra Review](https://github.com/cogine-ai/local-ultra-review), not a replacement.

| Skill | Purpose |
| --- | --- |
| Local Ultra Review | Generate high-confidence PR/code review findings |
| Loop on PR Review and Fix | Respond to existing PR review feedback with verified fixes |

Use Local Ultra Review when you need to produce a review. Use Loop on PR Review and Fix when reviewers or review bots have already commented and you need the PR branch to converge.

## Philosophy

Loop on PR Review and Fix is intentionally conservative:

**Verify first. Fix narrowly. Validate before pushing. Remember every decision.**

It is better to skip a vague comment with a clear reason than to ship a broad, unvalidated change just to make a review thread look handled.
