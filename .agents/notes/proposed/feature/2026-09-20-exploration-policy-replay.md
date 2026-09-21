# Agent Note: Recorded-run replay for exploration policies

Status: proposed

English | [中文](2026-09-20-exploration-policy-replay.zh.md)

## Problem

Long-horizon work in this harness is steered by strategies fixed at deployment. The `ralph` loop in `packages/workflow/tool-ralph/src/index.ts` is a deployment-owned script string the model may not alter; subagent fan-out is capped by `maxActiveSubagents` (default 8) for continuable activation in `packages/subagent/subagent/src/index.ts`; per-step tool batching is capped by `maxParallelToolCalls` (default 10) in `packages/core/agent-loop/src/constants.ts`; round pacing is bounded by `maxGoalRounds` in `packages/goal/goal/src/index.ts` and by ralph's `maxRounds`. Each of these answers the three questions an exploration policy answers: which piece of work continues next, how many run in parallel, and when to stop. None of them can improve from the history the harness already records.

Feedback for such a strategy is delayed and expensive. Judging a scheduling decision requires watching a whole run unfold, so comparing two candidate strategies costs one full rollout each. This harness therefore cannot answer "would a different batching or stopping rule have reached the same result for fewer agent calls?" without paying for the comparison.

Most of the material needed to answer it offline already exists. Every run is an append-only typed log ([session events](../../../../packages/core/session/src/types.ts)) made durable by [session-persistence-jsonl](../../../../packages/session/session-persistence-jsonl/README.md), carrying per-attempt settlements (`assistant/attempt`), turn and step boundaries, tool calls with timings, and delegation lineage. Four things are missing: a score attached to an attempt, a tree over attempts, an evaluator that traverses that tree without executing anything, and a seam where batching and stopping decisions are programmable rather than configured.

The mechanism proposed here is the one in *Dream-RSI: Recursive Self-Improvement through Evolving Worlds* (arXiv:2609.14858, `zhengkid/Dream-RSI`): a completed run is itself a replay simulator over the search space it reached, so thousands of candidate exploration policies can be scored offline at zero executions and only the winner redeployed. Its reference implementation is unreleased, and its Python policy contract (`see.policy.api`) belongs to a different runtime, so this proposal reimplements the mechanism on this harness's existing seams instead of adopting that runtime.

## Proposal

Four capabilities in dependency order. Each is useful on its own; A and B are useful before any policy is programmable.

### A. Attempt tree and score record

An attempt is one unit of work the exploration policy can choose: in this harness, one subagent run or one refinement round of a long-horizon loop. Each attempt becomes a node with a durable identity, a parent, the workspace state it started from, its outcome, and a score supplied by the task.

The node is recorded as session events, so it inherits the log's existing guarantees (contiguous `seq`, single writer, crash repair) instead of inventing a second store. Tree shape is a projection over those events, in the same style as the existing projection units in `packages/session/session-projection`.

Workspace capture is the expensive part and has no durable implementation today: [workspace-changes](../../../../packages/deliverables/workspace-changes/README.md) records turn-level git snapshots whose trees and copies live only as long as the session in that Host process, and `captureWorkspaceSnapshot` in `packages/test-support/session-snapshot/src/workspace.ts` is complete but test-only. A first version reuses the test-kit capture and stores content-addressed artifacts through the `storage` hub; it does not attempt incremental snapshots.

### B. Replay scorer

`ctx.explorationReplay` answers one question offline: given a recorded tree and a candidate policy, which nodes would that policy have revealed, in which batches, and what would its recorded outcomes have been? Replay reveals already-recorded children deterministically instead of generating new work, so it performs no model call and no tool execution. Scoring therefore costs disk reads, not agent calls.

The scorer reports the three components separately rather than one number, because they trade against each other:

```
quality      = max score over the revealed subtree
cost         = number of revealed non-root nodes (the attempts it would have paid for)
parallelism  = revealed nodes per decision round
```

The combined replay score follows the paper, with `beta1` pricing work and `beta2` rewarding useful batching:

```
V = quality - beta1 * cost + beta2 * cost / max(1, rounds)
```

The paper's appendix states an equivalent Pareto form (`pareto_reward = pareto_auc_lambda * parallel_penalty`, where `parallel_penalty` is the mean of `effective_sequential_rounds / total_probes` over a beta sweep). The first implementation ships the additive form and keeps `beta1`/`beta2` as configuration; a sweep is a caller-side loop over `beta`, not new machinery.

### C. Exploration-policy seam

A policy is code, not configuration: given the prefix it has observed, it returns the next batch of nodes and, at a round boundary, whether to continue. Following the [capability-seam rule](../../implemented/architecture/2026-06-13-capability-seams.md), this is a Service Definition (`ctx.explorationPolicy`) with the harness supplying one fixed provider that reproduces today's behavior, so mounting the seam changes nothing until a policy is selected.

Two properties are contractual rather than advisory. First, **prefix-only**: a policy may read only nodes already revealed, the baseline score, and structural metadata; unrevealed scores, known-optimal node ids, and absolute score targets are out of reach by construction, because replay only hands it revealed observations. Second, **monotone selection**: the deployed policy is itself a candidate in every improvement round, and the next deployed policy is the argmax over all evaluated revisions, so a revision is never adopted on evidence that scores worse than what it replaces.

A policy also plans the shape of the next run (`plan_grid` in the paper's contract): how many branches to open and how deep to refine each. In this harness that plan is what bounds a rollout's width and depth, replacing today's fixed `maxParallelToolCalls` for runs that opt in. It replaces nothing else: `maxGoalRounds` stays the session's outer round ceiling, a plan's `rounds` is validated against it, and a plan that exceeds it fails loud rather than bypassing the bound or silently clipping it — the same resolution this document gives every other deployment ceiling.

### D. Policy-improvement loop

An outer loop runs the current policy online, appends the completed tree to a history of recorded trees, then asks a policy-development agent to revise the policy code and scores every revision by replay over the whole history. The revision loop composes existing pieces: the development agent is a `subagent` or a `workflow` script, the candidate policy is evaluated in the sandbox that already exists for model-written plugins (`evaluateHostCode` in `packages/extensions/cordis-host-runner/src/sandbox.ts`), and redeployment uses the existing plugin lifecycle (`packages/boot/plugin-manager`) or a runtime-mutable settings section, as `maxParallelToolCalls` already is.

The pool the loop scores against is bounded, and the two retention knobs bound different things. Tree skeletons — node identity, parentage, outcome, score, round counts — are log-derived and small, and `workspaceRetention` never prunes them; what it prunes is the content-addressed capture artifacts, the disk-heavy part, per node. A tree whose artifacts were pruned is no longer replayable, so it leaves the simulator pool and is counted as excluded rather than disappearing quietly, and the loop records the pool size and the excluded count beside every score it produces. `poolDepth`, a separate improvement-loop knob, bounds how many completed trees stay replayable at once. The pool therefore shrinks only by explicit retention, never as a side effect of a per-run storage setting, and a shrinking pool is visible in the scored record.

Redeployment is the one step that must not be automatic in a shipped profile: a rewritten policy is model-authored code, so it lands as a candidate a human or an existing review flow approves, not as an auto-installed plugin.

## The mechanism restated for this codebase

| Dream-RSI term | Meaning here |
|---|---|
| Discovery tree | Recorded attempts of one long-horizon run, linked parent to child |
| Eligible set | The root plus every revealed leaf: where the next attempt may start |
| Batch | The nodes one decision round expands, bounded by the run's worker budget |
| Online rollout | The real run: attempts execute, cost is agent calls, outcomes are stochastic |
| Replay | Traversal of recorded children only: no model call, no tool execution, deterministic |
| Replay objective | Quality minus priced work, plus a batching bonus |
| Policy improvement | A development agent rewrites the policy code; every revision is scored by replay |
| Simulator pool | Every recorded tree from every previous round that is still replayable, all used to score each revision; pruned artifacts remove a tree from the pool and the exclusion is counted |

## Data model

A node is one attempt — one subagent run or one refinement round. The alternative considered and rejected on the evidence available today is a whole session as the node: that makes replay nearly free to build and nearly useless, because the decisions a policy must make live inside a session rather than between sessions. The choice is provisional and phase 1's own measurements validate it (see Phasing); every field below is written for it, and changing it changes the event shape rather than the scorer's contract.

| Field | Source | Meaning |
|---|---|---|
| `nodeId` | New event | Branded identity of one attempt, stable across replay |
| `parentId` | New event | The node whose workspace this attempt continued from |
| `workspaceRef` | New event | Content-addressed reference to the captured starting state |
| `outcome` | New event | Terminal status plus the diagnostic text an evaluator returned |
| `score` | Task-supplied | The quality signal; absent until a task declares one |
| `turn`, `step`, `sessionId` | Existing events | The recorded facts that link a node back to its run |
| `rounds`, `attempts` | Derived | Counts the replay objective prices, never stored as authority |

## Replay semantics and limits

- A policy can only be dreamt where history actually went: replay reveals recorded children and never synthesizes an outcome, so the simulator is exact over the realized search space and blind outside it.
- The monotone guarantee is replay-local. It holds for the replay score on the fixed history; it says nothing about live performance, and the paper itself warns that the replay ceiling is not a live stopping signal.
- Determinism requires node identity, not call order. The existing replay adapter binds live sessions to recorded scripts by first-call order and documents that concurrent sibling branches bind non-deterministically, so a tree that admits concurrent branches must key every node by `nodeId`.
- Replay does not re-run the loop. `llm-replay` substitutes model streams but still executes the real loop and real tools; it is a keyless test adapter, not a simulator, and is not reused as the scorer.

## Mapping onto existing packages

| Piece | Existing surface to reuse | New work |
|---|---|---|
| Attempt nodes | Session log and persistence seam | New event types, merged into `SessionEventMap` |
| Tree projection | `packages/session/session-projection` | One projection unit folding attempt events |
| Workspace capture | `captureWorkspaceSnapshot`, `storage` hub | Durable, content-addressed, per-attempt storage |
| Replay scorer | `packages/core/session` folds, `session-query` reads | The scorer and its objective |
| Policy seam | Capability-seam pattern, `cordis.yml` config | Service Definition, one behaviour-preserving provider |
| Policy code evaluation | `evaluateHostCode` sandbox | Scoring harness around it |
| Redeploy | Plugin manager, runtime-mutable settings | Selection rule and review gate |

## Scoring is task-owned

This harness has no grader, judge, rubric, or quality metric of any kind; `benchmarks/` measures milliseconds and memory and is explicitly forbidden from consuming recorded sessions. The proposal therefore treats the score as an input the task owns, delivered through a declared evaluator seam, and keeps every quality metric out of the harness core. Inventing a proxy score inside the harness would optimize the proxy, and a self-improving loop makes that failure permanent rather than transient.

## Choosing `beta1` and `beta2`

The objective is imported from the paper, and it is the one knob that decides what "better" means, so its defaults carry a stated story instead of a bare number. Quality is normalized against the best recorded score in the pool when the components are combined, which turns `V` into "fraction of the best known outcome, minus priced work, plus a batching bonus"; the scorer still reports the raw components, so a task that knows its quality scale can price `beta1` against that scale directly.

The shipped defaults are `beta1 = 0.01` and `beta2 = 0.005`: a hundred revealed attempts cost as much as reaching the best known outcome, and batching can offset at most half the priced work. `beta2` must stay below `beta1` — at or above it the cost term is net-positive and the argmax is whatever reveals the most nodes regardless of quality, the opposite of the intent — so config validation rejects `beta2 >= beta1` rather than documenting it.

The other degenerate shapes are named so a sweep can recognize them. `beta1 = 0` makes revealing everything free, so the argmax collapses to maximum quality with no efficiency claim. `beta1` large relative to the spread of recorded scores makes the shortest rollout win, and since the deployed policy is itself a candidate, "change nothing" then wins every round forever.

Two rules keep those shapes visible rather than silent. A sweep reports the argmax at each `beta`, so a decision that exists at only one `beta` is visible as such. And a revision whose revealed tree is empty is invalid rather than selected: monotone selection compares scored revisions, and a policy that reveals nothing has not scored anything.

## Configuration surface

| Knob | Owner | Purpose |
|---|---|---|
| `beta1` | Replay scorer config | Price of one attempt in the replay objective |
| `beta2` | Replay scorer config | Weight of the batching bonus |
| `maxParallelism` | Policy plan, bounded by config | Workers per decision round |
| `branchCount`, `refineCount` | Policy plan, bounded by config | Width and depth of the next run |
| `rounds`, `revisions` | Improvement-loop config | Outer iterations and policy revisions per iteration |
| `workspaceRetention` | Storage config | How many nodes' capture artifacts a run keeps; pruning drops artifacts, never tree structure or scores |
| `poolDepth` | Improvement-loop config | How many completed trees stay replayable in the simulator pool |

Every knob is a validated `Config` field changeable from `cordis.yml`, per the no-hardcoded-tunables rule; the deployment ceiling stays fixed and a plan that exceeds it fails loud rather than silently clipping.

## Phasing

1. **Score and tree** — one long-horizon workload records attempt nodes with a task-supplied score, restorable after restart. This phase also settles node granularity provisionally (see Data model): it reports per-attempt capture cost and the breadth of the recorded tree, and the choice is revisited if those measurements contradict it.
2. **Replay scorer** — the recorded tree is scored offline; the deployed policy's own replay reproduces its recorded trajectory.
3. **Offline comparison** — the fixed strategy is compared against variants with no policy seam and no rewrite loop, which already answers whether a cheaper batching rule existed. A negative answer is a legitimate end of the effort: the project stops with a measured, simulated comparison and no deployable win, and phase 4 is not entered.
4. **Policy seam** — batching, width, depth, and stopping become programmable behind one Service Definition.
5. **Improvement loop** — revisions are generated, scored by replay, and redeployed under review, with each redeployment gated on the live-versus-replay divergence check.

## Alternatives considered

### Why not use `llm-replay` and the snapshot harness as the simulator?

They replay recorded model streams into a real loop that still executes real tools and real workspaces, so they are not zero-execution and cannot score a policy that would have taken different branches; their script binding is also first-call-order, which the package documents as non-deterministic under concurrent sibling delegation. They remain the right tool for their purpose, keyless regression of shipped profiles, and the wrong tool for counterfactual scoring.

### Why not build the tree from session `fork()` lineage?

`fork` clips at stable boundaries of live sessions, creates a single child per boundary, and the session README defers a deeper entry tree explicitly. That is a lineage model for display and resumption, not a frontier a policy selects from: it has no node identity, no per-attempt score, and no way to enumerate sibling attempts at one decision point.

### Why not tune the fixed knobs by grid search over live runs?

That is exactly the expensive online path this design exists to avoid: each candidate costs a full rollout, feedback arrives only after the run ends, and the meta-space is large enough that most candidates are bad. Replay makes the comparison free of agent calls, so the online budget is spent only on the winner.

### Why not adopt the paper's implementation directly?

The repository is unreleased, and its policy contract is Python against a foreign `see.policy.api` runtime with its own trace pool and grid planner. Adopting it would add a second runtime beside the harness's own persistence, projection, sandbox, and plugin lifecycle, and would duplicate recording the harness already performs.

### Why not steer exploration with prompt guidance instead of replay?

The paper measures this directly: injecting abstracted directional insights into the prompt underperforms unguided exploration at equal budget on the task where it was tested, because strong priors over-constrain a multi-threaded search. This harness already carries a large amount of guidance, so adding more is both the cheaper instinct and the measured-worse one.

### Why not let the model schedule through the agent-team task board?

Model-claimed tasks in `packages/experimental/agent-team` are chosen by the model at runtime and leave no scorable decision record; replay needs the batching and stopping decision to be code that can be replayed against the same tree, and needs it to be reproducible for the same prefix.

## Acceptance criteria

- One long-horizon run produces a tree whose nodes carry identity, parent, workspace reference, outcome, and score, and the tree is restorable from persistence after a restart with identical parentage.
- `ctx.explorationReplay` scores a candidate policy over at least two recorded trees with zero model calls and zero tool executions, deterministically: the same policy and trees yield the same score, and the three components are reported separately.
- Replaying the currently deployed policy reproduces its own recorded trajectory, node for node, which is the self-consistency check that the recorded tree is complete.
- The selection rule holds mechanically: a revision that scores below the deployed policy is never selected, and the deployed policy is always among the evaluated candidates.
- A policy cannot read an unrevealed score, a hardcoded winning node id, or an absolute score target; the constraint is enforced by what replay exposes, not by review alone.
- Every new knob is a validated `Config` field changeable from `cordis.yml`; no `DEFAULT_*` constant substitutes for configurability.
- A rewritten policy reaches a shipped profile only through an explicit review step, never as an automatic install.
- A redeployed policy's first live rounds are compared against the trajectory its own replay predicted; divergence beyond a configured tolerance flags the round and pauses redeployment, and the reviewing human sees that divergence report rather than only the code diff.
- Every recorded score states the simulator pool it was computed over, including how many recorded trees were excluded because they are no longer replayable.
- Documentation, bilingual pairs, and the recorded-session snapshots required by the testing policy land in the same change.

## Risks

- **No score exists yet.** The whole mechanism is blocked on a task-owned quality signal. If the project invents a convenient proxy inside the harness, a self-improving loop will optimize the proxy and lock that in.
- **Node granularity is provisional.** The paper's node is one cheap generation-and-evaluation; a session here is expensive and long. This document records one attempt per node, because session-granularity nodes make replay cheap to build and nearly useless while attempt-granularity nodes are useful and require per-attempt workspace capture; phase 1 reports per-attempt capture cost and tree breadth to confirm or refute that, and the fallback is a coarser unit — one decision round rather than one attempt — at the cost of replay resolution.
- **Workspace capture costs disk.** One capture per attempt, content-addressed, is new storage growth the harness does not have today; retention prunes those artifacts only, and the tree skeleton and scores survive pruning (see D).
- **Realized trees are narrow.** Fan-out caps of 8 subagents and 10 parallel tool calls bound how much of the search space any recorded tree covers; widening them changes product behaviour and is not part of this proposal.
- **Monotone replay is not monotone live.** The guarantee is over fixed history only, so a selected revision can still regress a live run.
- **Model-written policy code is a security surface.** Executing and installing rewritten policy code needs the existing sandbox and an explicit review gate; auto-install into a shipped profile is out of scope.
- **Replay ceiling misread as a stopping signal.** A policy tuned to the recorded ceiling stops early on live runs where the ceiling is not the live optimum; the paper reports this as a live failure mode, not a theoretical one.

## Open questions

- Does phase 1 confirm one attempt per node, and if per-attempt capture cost is unaffordable, how much replay resolution does the coarser decision-round fallback lose?
- Who declares the score: a task configuration, a tool the model calls, or an evaluator plugin the deployment mounts?
- Should the policy decide batching only across attempts, or also across tool calls inside one step, where `executeToolCalls` already runs a bounded pool?
- Do retry settlements (`assistant/attempt`, `llm/retry`) become nodes, or stay attributed to the attempt that contains them?
- Does the first implementation live in `packages/experimental/` until the seam is proven, and which shipped profile is the first to opt in?
- Should `maxGoalRounds` be exposed to the policy as part of its observed prefix, so a plan can size itself to the remaining budget?
