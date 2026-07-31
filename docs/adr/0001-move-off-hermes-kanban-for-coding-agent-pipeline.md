# ADR 0001: Move off Hermes Kanban as the driver for the coding agent pipeline

- **Status:** Accepted
- **Date:** 2026-07-31
- **Author:** danielhendrickson.guest (with Drover/Hermes assist)

## Context

Hermes Kanban (`hermes kanban`) has been used as the dispatch mechanism for
coding-agent work: cards assigned to profiles (dev/review/test/research),
picked up via skills like `kanban-doer`, each card driving an agent through
some version of clone → work → commit → PR.

Overnight, this broke down. The core problem: **agent profiles working
Kanban cards did not reliably execute the full pipeline every time.** In
practice this has meant, across different runs:

- Work sometimes wasn't pushed before the scratch workspace was garbage
  collected (lost tickets — this happened at least once before a
  `kanban-doer` skill fix required a git push before marking a card done).
- Whether a VM got spun up fresh, whether the repo got cloned, whether the
  work got committed, and whether a PR got opened were all *behaviorally
  emergent* from the agent's judgment in the moment, not *structurally
  guaranteed* by the pipeline. A profile "usually" did the right thing;
  it did not *always* do the right thing.
- Diagnosing failures required after-the-fact reconstruction (was there a
  card? did the workspace still exist? did the branch get pushed?) rather
  than reading a deterministic execution log.

This is a fundamental mismatch: Kanban + agent-profile dispatch is a
*task-tracking and routing* system. It was being asked to also be a
*deterministic execution pipeline* — a role it wasn't designed for and
doesn't guarantee.

## Decision

**Stop using Hermes Kanban + ad hoc agent profiles as the mechanism that
drives the actual coding-agent execution pipeline** (VM lifecycle → clone →
work → commit → PR).

Kanban may still be useful as a *task board / status tracker* layered on
top of a real pipeline, but it is no longer trusted as the thing that
*causes* the pipeline steps to happen. The pipeline itself needs to be a
repeatable, inspectable process where every run does the same fixed set of
steps in the same order, regardless of what the agent decides in the
moment.

### Required properties of the replacement

For any coding task dispatched to an agent, the pipeline must, every time,
without relying on agent judgment to remember or decide to do it:

1. **Spin up a new VM** — fresh, disposable, no leftover state from a
   prior run.
2. **Clone** the target repo into that VM.
3. **Do the work** (the actual agent task — this step's *content* varies,
   but its occurrence does not).
4. **Commit the work** — always, even if the task terminates early, stalls,
   or times out, so partial work isn't silently lost.
5. **Open a PR if appropriate** — the "if appropriate" gate should be an
   explicit, checkable rule (e.g., "there is a commit that isn't on a
   protected branch, and the target branch is not `main`"), not a judgment
   call the agent might skip.

This points toward the pipeline steps being enforced *outside* the agent's
own discretion — e.g., a wrapper/orchestrator script or cron job that
performs steps 1, 2, 4 (commit) and 5 (PR-open check) mechanically, and
only delegates step 3 (the actual work) to the agent. The agent should not
be the thing responsible for remembering to clone, commit, or open a PR —
it should be handed a workspace that's already cloned, and its own
process/skill exit should trigger a commit+PR check regardless of how the
agent's own turn ended.

## Consequences

- We lose Kanban's card-based visibility/UI for "what's queued, what's in
  progress" unless we deliberately re-add a tracking layer on top of the
  new pipeline.
- We need to design and build the actual deterministic runner (VM
  lifecycle, clone, commit-on-exit, PR-gate). This ADR does not specify
  that design — it only records the decision to stop relying on Kanban +
  agent profiles for it, and the required properties above.
- Existing `kanban-doer`/`kanban-reviewer`/etc. skills and profile roster
  remain around for other uses but are no longer the source of truth for
  "did the coding pipeline actually run."
- Any in-flight Kanban cards depending on this dispatch model should be
  audited before this cutover to make sure nothing is silently abandoned.

## Alternatives considered

- **Patch Kanban further** (e.g., stricter skill preconditions, more
  cron-based verification). Rejected for now: the failure mode is
  structural (no enforcement layer outside agent judgment), not a specific
  bug to patch. Repeated patching has already happened once (the git-push
  requirement) and the underlying reliability gap remains.
- **Keep Kanban for tracking, build enforcement around it.** Not rejected
  outright — likely revisited once the deterministic runner exists, as a
  way to get visibility back without reintroducing the reliability gap.
