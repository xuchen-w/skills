---
name: simplify
description: Review changed code for reuse, simplification, efficiency, and altitude issues, then apply behavior-preserving fixes. Use when asked to simplify or clean up a diff, branch, PR, or changed files, including requests for Claude Code-style simplify. This skill performs the workflow in the current agent; it does not invoke Claude Code or conduct a correctness/security review.
---

# Simplify

Adapted from the bundled `/simplify` prompt extracted and request-verified from
Claude Code 2.1.278. The workflow below preserves the original parallel prompt;
the following host adaptations supply the tool mapping and fallback behavior.

## Host adaptations

- Run this workflow using the current agent and its host's native subagent
  facilities. `Agent` means the host's spawn/delegate tool; `Grep` means its code
  search facility. In Codex, use the available subagent spawn, message, and wait
  tools. Do not invoke the Claude Code CLI or create separate user-owned tasks.
- This skill explicitly requests delegation to four independent review agents.
  Use general-purpose review-capable subagents; no custom agent definitions,
  model overrides, or reasoning-effort overrides are required. Inherit the
  session defaults unless the user explicitly specifies otherwise.
- Give each reviewer the same diff and target, its assigned angle, the applicable
  repository instructions and constraints, and the findings format below.
  Reviewers may inspect surrounding code for evidence, but return findings
  without editing files or delegating further. The parent applies the fixes
  after gathering the reviews.
- Dispatch all four reviews before waiting when the host permits. If the host
  requires separate spawn calls, issue them consecutively without waiting for
  each review to finish. If capacity is lower than four, run the four independent
  reviews in batches and report that scheduling difference. Do not omit angles.
- If subagents are unavailable, disallowed by the host, or blocked by the nesting
  depth limit, use the single-pass fallback below. A reviewer error is not a clean
  review: complete its angle inline if possible and disclose the partial fallback;
  otherwise report that angle as incomplete.
- Interpret any user-specified target as the review target in Phase 0.
  `/code-review` below names the separate correctness-review workflow; it is not
  an instruction to invoke a command that may not exist on this host.

## Single-pass fallback

When the four-agent workflow cannot run, replace the launch instructions in
Phase 1 with the following:

Work through all four angles yourself, in this same context, in one pass — do not
skip an angle for lack of fan-out. For each, note findings with `file`, `line`, a
one-line `summary`, and the concrete cost (what is duplicated, wasted, or harder
to maintain).

Then perform Phase 2 without waiting for subagents. State clearly in your summary
that this was a single-pass review done without the Agent tool, not the full
4-agent fan-out, so whoever reads it is not misled about what actually ran.

## Original parallel workflow

`/simplify → 4 cleanup agents in parallel → apply the fixes`

You are improving the quality of the changed code, not hunting for bugs. Review
it for reuse, simplification, efficiency, and altitude issues, then fix what you
find. Do not look for correctness bugs — that is what `/code-review` is for.

## Phase 0 — Gather the diff

Run `git diff @{upstream}...HEAD` (or `git diff main...HEAD` / `git diff HEAD~1`
if there's no upstream) to get the unified diff under review. If there are
uncommitted changes, or the range diff is empty, also run `git diff HEAD` and
include the working-tree changes in scope — the review often runs before the
commit. If a PR number, branch name, or file path was passed as an argument,
review that target instead. Treat this diff as the review scope.

## Phase 1 — Review (4 cleanup agents in parallel)

Launch **4 independent review agents** via the Agent tool, all in a
single message so they run concurrently. Pass each agent the diff and one of
the four angles below. Each returns its findings with `file`, `line`, a
one-line `summary`, and the concrete cost (what is duplicated, wasted, or
harder to maintain).

### Reuse

Flag new code that re-implements something the codebase
already has — Grep shared/utility modules and files adjacent to the change,
and name the existing helper to call instead.

### Simplification

Flag unnecessary complexity the diff adds: redundant or derivable state,
copy-paste with slight variation, deep nesting, dead code left behind. Name
the simpler form that does the same job.

### Efficiency

Flag wasted work the diff introduces: redundant computation or repeated I/O,
independent operations run sequentially, blocking work added to startup or
hot paths. Also flag long-lived objects built from closures or captured
environments — they keep the entire enclosing scope alive for the object's
lifetime (a memory leak when that scope holds large values); prefer a
class/struct that copies only the fields it needs. Name the cheaper
alternative.

### Altitude

Check that each change fixes the root cause at the right depth rather than
patching a symptom with a fragile bandaid. Special cases layered on shared
infrastructure are a sign the fix isn't deep enough — prefer the simpler, more
general change to the underlying mechanism over adding special cases, and name
that change.

## Phase 2 — Apply the fixes

Wait for all four agents to complete, dedup findings that point at the same
line or mechanism, and fix each remaining one directly. Skip any finding whose
fix would change intended behavior, require changes well outside the reviewed
diff, or that you judge to be a false positive — note the skip rather than
arguing with it. Finish with a brief summary of what was fixed and what was
skipped (or confirm the code was already clean).
