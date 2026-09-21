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

```
Recorded trees ──► replay evaluator ──► quality | cost | parallelism
                                            │
                    objective sweep + candidate comparison
                                            │
                     replay monotonicity over recorded history
                                            │
           live shadow rounds ──► replay fidelity vs prediction
                                            │
                                   human deployment
```

### A. Attempt tree, decision rounds, and score record

An attempt is one unit of work the exploration policy can choose: in this harness, one subagent run or one refinement round of a long-horizon loop. Each attempt becomes a node with a durable identity, a parent, the workspace state it started from, its outcome, and a score supplied by the task.

The node is recorded as session events, so it inherits the log's existing guarantees (contiguous `seq`, single writer, crash repair) instead of inventing a second store. Tree shape is a projection over those events, in the same style as the existing projection units in `packages/session/session-projection`.

A policy decision is a first-class record rather than an inference from node parentage. Each decision round records what the policy observed (the eligible frontier), which nodes it selected, the parallelism it asked for, whether it continued or stopped, and the nodes that resulted — so "these four attempts were competing alternatives at one decision" is recorded fact, not a reconstruction. An attempt belongs to the round that launched it. The baseline provider records rounds before any policy exists, because the fixed strategy makes these decisions too.

Workspace capture is the expensive part and has no durable implementation today: [workspace-changes](../../../../packages/deliverables/workspace-changes/README.md) records turn-level git snapshots whose trees and copies live only as long as the session in that Host process, and `captureWorkspaceSnapshot` in `packages/test-support/session-snapshot/src/workspace.ts` is complete but test-only. A first version reuses the test-kit capture and stores content-addressed artifacts through the `storage` hub; it does not attempt incremental snapshots.

### B. Replay scorer

`ctx.explorationReplay` answers one question offline: given a recorded tree and a candidate policy, which nodes would that policy have revealed, in which batches, and what would its recorded outcomes have been? Replay reveals already-recorded children deterministically instead of generating new work, so it performs no model call and no tool execution. Scoring therefore costs disk reads, not agent calls.

The scorer reports the three components separately rather than one number, because they trade against each other:

```
quality      = max score over the revealed subtree
cost         = number of revealed non-root nodes (the attempts it would have paid for)
parallelism  = revealed nodes per decision round
```

Alongside the three components the scorer emits the trajectories they are computed from, so a later objective choice never needs the history re-read: `best_score_by_round`, `best_score_by_cost`, `rounds`, and `quality_at_budget` (the best score reached within a stated cost). The combined `V` is derived from those artifacts and is never the only thing kept.

The combined replay score follows the paper, with `beta1` pricing work and `beta2` rewarding useful batching:

```
V = quality - beta1 * cost + beta2 * cost / max(1, rounds)
```

The paper's appendix states an equivalent Pareto form (`pareto_reward = pareto_auc_lambda * parallel_penalty`, where `parallel_penalty` is the mean of `effective_sequential_rounds / total_probes` over a beta sweep). The first implementation ships the additive form and keeps `beta1`/`beta2` as configuration; a sweep is a caller-side loop over `beta`, not new machinery.

### C. Exploration-policy seam

A policy is code, not configuration: given the prefix it has observed, it returns the next batch of nodes and, at a round boundary, whether to continue. Following the [capability-seam rule](../../implemented/architecture/2026-06-13-capability-seams.md), this is a Service Definition (`ctx.explorationPolicy`) with the harness supplying one fixed provider that reproduces today's behavior, so mounting the seam changes nothing until a policy is selected.

A policy does not receive the harness context. Its contract is `policy(observation) -> action`, where `PolicyObservation` carries the round index, the observed frontier, the revealed attempts with their scores and outcomes, the remaining budget, and structural metadata, and nothing else. The observation is deeply frozen and the returned action is validated against the frontier before it is recorded, so hidden nodes, unrevealed scores, evaluator internals, and live references into the scorer are unreachable by construction rather than by convention.

Two properties are contractual rather than advisory. First, **prefix-only**: a policy may read only nodes already revealed, the baseline score, and structural metadata; unrevealed scores, known-optimal node ids, and absolute score targets are out of reach by construction, because replay only hands it revealed observations. Second, **purity**: a policy is a pure function of its observation — no mutation, no filesystem, no network, no evaluator access, no wall-clock dependence, and no unseeded randomness — and the scoring harness re-runs it on the same observation to detect nondeterminism or mutation rather than trusting the declaration. Third, **monotone selection**: the deployed policy is itself a candidate in every improvement round, and the next deployed policy is the argmax over all evaluated revisions, so a revision is never adopted on evidence that scores worse than what it replaces.

A policy also plans the shape of the next run (`plan_grid` in the paper's contract): how many branches to open and how deep to refine each. In this harness that plan is what bounds a rollout's width and depth, replacing today's fixed `maxParallelToolCalls` for runs that opt in. It replaces nothing else: `maxGoalRounds` stays the session's outer round ceiling, a plan's `rounds` is validated against it, and a plan that exceeds it fails loud rather than bypassing the bound or silently clipping it — the same resolution this document gives every other deployment ceiling.

### D. Policy-improvement loop

An outer loop runs the current policy online, appends the completed tree to a history of recorded trees, then asks a policy-development agent to revise the policy code and scores every revision by replay over the simulator pool (see The simulator pool). The revision loop composes existing pieces: the development agent is a `subagent` or a `workflow` script, the candidate policy is evaluated in the sandbox that already exists for model-written plugins (`evaluateHostCode` in `packages/extensions/cordis-host-runner/src/sandbox.ts`), and redeployment uses the existing plugin lifecycle (`packages/boot/plugin-manager`) or a runtime-mutable settings section, as `maxParallelToolCalls` already is.

Retention and compatibility decide which recorded trees may score a revision, and both are specified in The simulator pool below.

Redeployment is the one step that must not be automatic in a shipped profile: a rewritten policy is model-authored code, so it lands as a candidate a human or an existing review flow approves, not as an auto-installed plugin.

Replay fidelity is a metric, not a boolean. Each live round reports selection agreement, batch-size agreement, stop-decision agreement, outcome availability, score consistency, and cost agreement against what replay predicted, and divergence past the configured tolerance pauses redeployment. The human approving a candidate therefore sees the fidelity report and not only the code diff; divergence is the signal that the history a revision was selected on no longer describes the deployment.

## Contractual invariants

Four properties are the design's load-bearing claims, and each is enforced by what the implementation exposes rather than by review alone.

1. **Realized-tree counterfactual only.** Replay evaluates policies over the search tree a run actually realized. It estimates the value of alternative selection orders among recorded nodes; it does not estimate outcomes of branches that were never executed. A winning policy is better over recorded history, never proven optimal over the space the run could have reached.
2. **Prefix-only observation.** A policy sees the frontier it observed and nothing else: unrevealed scores, known-optimal ids, absolute targets, and evaluator internals are unreachable by construction.
3. **Policy purity.** Observation in, action out: no mutation of the observation, no ambient state, no unseeded nondeterminism.
4. **Replay monotonicity, then live validation.** `V(P_new | H) >= V(P_current | H)` before deployment, and `Live(P_new) ≈ Replay(P_new | H)` within a configured tolerance afterwards. Selection is proven on history; deployment is justified by fidelity, not by the selection proof.

## The mechanism restated for this codebase

| Dream-RSI term | Meaning here |
|---|---|
| Discovery tree | Recorded attempts of one long-horizon run, linked parent to child |
| Eligible set | The root plus every revealed leaf: where the next attempt may start |
| Batch | The nodes one decision round expands, bounded by the run's worker budget |
| Decision round | One policy decision: the observed frontier, the selected subset, the parallelism asked for, and the stop/continue choice |
| Online rollout | The real run: attempts execute, cost is agent calls, outcomes are stochastic |
| Replay | Traversal of recorded children only: no model call, no tool execution, deterministic |
| Replay objective | Quality minus priced work, plus a batching bonus |
| Policy improvement | A development agent rewrites the policy code; every revision is scored by replay |
| Simulator pool | Every recorded tree from every previous round that is still replayable, all used to score each revision; pruned artifacts remove a tree from the pool and the exclusion is counted |

## Data model

Two units are recorded, and separating them is what answers the granularity question this document previously left open. A node is one attempt — one subagent run or one refinement round — and it is the unit replay reveals and the objective prices. A decision round is the unit the policy acts in: it observes a frontier, selects a subset, sets parallelism, and continues or stops. Attempts are what a policy chooses; rounds are when it chooses, and recording the round is what keeps selection, non-selection, and parallelism as facts rather than derived behaviour.

What stays provisional is narrower than before: whether a subagent run and a refinement round are comparable enough to share one cost unit is measured in phase 1, and the fallback is to price nodes by recorded cost rather than by count.

| Field | Source | Meaning |
|---|---|---|
| `nodeId` | New event | Branded identity of one attempt, stable across replay |
| `parentId` | New event | The node whose workspace this attempt continued from |
| `workspaceRef` | New event | Content-addressed reference to the captured starting state |
| `outcome` | New event | Terminal status plus the diagnostic text an evaluator returned |
| `score` | Task-supplied | The quality signal; absent until a task declares one |
| `turn`, `step`, `sessionId` | Existing events | The recorded facts that link a node back to its run |
| `rounds`, `attempts` | Derived | Counts the replay objective prices, never stored as authority |
| `decisionRoundId` | New event | The decision round that launched this attempt |
| `eligibleAtRound` | New event (round) | The frontier the policy observed before selecting |
| `selectedAtRound` | New event (round) | The subset the policy chose; the complement is the recorded non-selection |
| `policyId`, `policyHash`, `policyVersion` | New event (round) | Immutable identity of the policy that decided |
| `sequenceWithinRound` | Derived | Position of a node inside its round's batch |
| `replayable` | Tree metadata | Whether artifacts survive retention and the fingerprint is compatible |

## Replay semantics and limits

- A policy can only be dreamt where history actually went: replay reveals recorded children and never synthesizes an outcome, so the simulator is exact over the realized search space and blind outside it (see Contractual invariants 1). Recorded outcomes also age: a tree can stay structurally replayable while the model, prompt, tool configuration, or environment that produced its scores has moved on, so staleness is expected to be the dominant failure mode rather than an edge case (see The simulator pool).
- The monotone guarantee is replay-local. It holds for the replay score on the fixed history; it says nothing about live performance, and the paper itself warns that the replay ceiling is not a live stopping signal.
- Determinism requires node identity, not call order. The existing replay adapter binds live sessions to recorded scripts by first-call order and documents that concurrent sibling branches bind non-deterministically, so a tree that admits concurrent branches must key every node by `nodeId`.
- Replay does not re-run the loop. `llm-replay` substitutes model streams but still executes the real loop and real tools; it is a keyless test adapter, not a simulator, and is not reused as the scorer.

## Replay state contract

Replay never re-executes, so this contract is not about restoring state; it decides which recorded outcomes may be treated as evidence for a policy evaluated today. A replay is valid for a candidate only where the factors below match between the recorded run and the deployment the policy would run in.

| Factor | Recorded today | Restored on replay |
|---|---|---|
| Workspace contents | `workspaceRef` per attempt (new work) | Yes, the only per-attempt capture |
| Model, provider, sampling | `request/header` epoch `config` (`LlmCallConfig`: `model`, `provider`, `maxTokens`, `temperature`, `reasoningEffort`) | No, identity only |
| Tool set and definitions | `request/header` epoch `header.tools`, session settings | No, identity only |
| Prompt composition | `system/message` events plus the epoch's `systemPromptUpdate` marker | No, identity only |
| Working directory, agent preset | `SessionHeader.cwd`, `SessionHeader.agentPreset` | No, identity only |
| Environment and installed packages | Not recorded | No |
| Seeds and randomness sources | Not recorded, except where a component declares one | No |

The per-attempt capture is therefore the one expensive artifact, and everything else is an identity a fingerprint can hash. A mismatch does not invalidate the tree as a record: it disqualifies its scores as evidence for a policy that would run under the current configuration, which is a stronger claim than "the tree is still replayable".

## The simulator pool: retention and compatibility

The pool the loop scores against is bounded, and the two retention knobs bound different things. Tree skeletons — node identity, parentage, outcomes, scores, decision-round records — are log-derived and small, and `workspaceRetention` never prunes them; what it prunes is the content-addressed capture artifacts, the disk-heavy part, per node. A tree whose artifacts were pruned is no longer replayable, so it leaves the pool and is counted as excluded rather than disappearing quietly, and the loop records the pool size and the excluded count beside every score it produces. `poolDepth`, a separate improvement-loop knob, bounds how many completed trees stay replayable at once.

Every tree also carries an explicit `replayable` flag and a compatibility fingerprint: model and version, harness commit, policy hash, prompt and configuration hashes, tool-configuration hash, and an environment fingerprint. The pool partitions into compatible trees (fingerprint matches the deployment being scored), stale trees (replayable, but produced under a different model, prompt, tool configuration, or environment), and incompatible trees (artifacts pruned, or the fingerprint cannot be compared). The scorer never mixes partitions silently: it reports which partition each candidate was scored over, and the improvement loop may not select a revision on stale-only evidence. This harness is a developer preview, so fingerprints are expected to diverge often and the partition is a working part of the design rather than a precaution.

The pool therefore shrinks only by explicit retention, never as a side effect of a per-run storage setting, and a shrinking pool is visible in the scored record.

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

## How the scorer aggregates task scores

`quality` as the maximum score in the revealed subtree is the paper's choice, and it is not neutral: a maximum rewards lottery-ticket exploration, because a policy that pays for ninety-nine useless attempts is fully compensated by one excellent node. For a harness whose attempts are expensive agent runs that bias is a decision rather than a default, so the aggregation is a validated scorer setting with four shipped options:

| `qualityDefinition` | Value | Bias |
|---|---|---|
| `max` | Best score anywhere in the revealed subtree | Lottery tickets: cost stops mattering once one node is excellent |
| `topKMean` | Mean of the best `k` revealed scores | Tolerates a few wins without collapsing to one |
| `bestAtBudget` | Best score reachable within a stated cost | Makes the budget explicit instead of leaving it implicit in `beta1` |
| `areaUnderBestCurve` | Area under best-score-versus-cost | Separates "reaches 95 in 20 attempts" from "reaches 95 in 50", which `max` collapses |

`max` stays the default because it is the measured form, and phase 3 decides whether it survives contact with real runs; the trajectory artifacts above are what let that decision be made without re-reading history.

## Choosing `beta1` and `beta2`

The objective is imported from the paper, and it is the one knob that decides what "better" means, so its defaults carry a stated story instead of a bare number. Quality is normalized against the best recorded score in the pool when the components are combined, which turns `V` into "fraction of the best known outcome, minus priced work, plus a batching bonus"; the scorer still reports the raw components, so a task that knows its quality scale can price `beta1` against that scale directly.

The shipped defaults are `beta1 = 0.01` and `beta2 = 0.005`: a hundred revealed attempts cost as much as reaching the best known outcome, and batching can offset at most half the priced work. The effective marginal cost of a revealed node is `-beta1 + beta2 / rounds`, so the batching bonus cancels the price of work exactly in single-round policies: at `rounds = 1` the bonus is `beta2` per node, and `beta2 >= beta1` makes revealing nodes free or profitable, with the argmax becoming whatever reveals the most nodes regardless of quality. Because that bonus is largest at `rounds = 1` and only diluted afterwards, `beta2 < beta1` is exactly the condition that the marginal cost stays positive at every round count, and config validation enforces it rather than documenting it. A deployment that wants a different tradeoff moves `beta1`, not the sign of the cost.

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
| `qualityDefinition` | Replay scorer config | How task scores aggregate into `quality` (`max`, `topKMean`, `bestAtBudget`, `areaUnderBestCurve`) |
| `fidelityTolerance` | Improvement-loop config | Divergence allowed between a live round and its replay prediction before redeployment pauses |
| `workspaceRetention` | Storage config | How many nodes' capture artifacts a run keeps; pruning drops artifacts, never tree structure or scores |
| `poolDepth` | Improvement-loop config | How many completed trees stay replayable in the simulator pool |

Every knob is a validated `Config` field changeable from `cordis.yml`, per the no-hardcoded-tunables rule; the deployment ceiling stays fixed and a plan that exceeds it fails loud rather than silently clipping.

## Phasing

1. **Score, tree, and decision rounds** — one long-horizon workload records attempt nodes with a task-supplied score, restorable after restart, and records the decision rounds the baseline strategy made along with them. This phase also settles the one open pricing question (see Data model): it reports per-attempt capture cost and the breadth of the recorded tree, and prices nodes by recorded cost rather than by count if those measurements contradict the attempt unit.
2. **Replay scorer** — the recorded tree is scored offline; the deployed policy's own replay reproduces its recorded trajectory.
3. **Offline comparison (gate)** — with no policy seam and no rewrite loop, the recorded tree is scored for six strategies that must come apart: the deployed fixed strategy, breadth-first, depth-first, random, greedy-best-score, and cost-aware. The gate is met when the deployed strategy's replay reproduces its recorded decisions round for round and the alternatives rank in an order a human can defend. A failure is a legitimate end of the effort: the project stops with a measured, simulated comparison and no deployable win, and phases 4 and 5 are not started.
4. **Policy seam** — batching, width, depth, and stopping become programmable behind one Service Definition.
5. **Improvement loop** — revisions are generated, scored by replay, and redeployed under review, with each redeployment gated on the live-versus-replay divergence check.

## Alternatives considered

### Why not learn a simulator for branches that were never executed?

A learned model of unexecuted branches would answer the stronger counterfactual, and it is a different project: it needs its own training signal, it inserts a proxy whose errors the optimizer will exploit, and the paper's own guarantee is replay-local. Recorded-tree replay is the honest version of the claim; a learned simulator would have to be argued on its own evidence, and this document does not claim it.

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
- Every decision round is recorded with the frontier it observed, the subset it selected, its parallelism, its stop decision, and the nodes that resulted, so competing alternatives at one decision are recorded fact rather than inference, and the deployed strategy's own decisions replay round for round.
- A policy receives a deeply frozen `PolicyObservation` and returns a validated action; it cannot reach hidden nodes, unrevealed scores, evaluator internals, ambient state, or an unseeded randomness source, and re-running it on the same observation yields the same action.
- The scorer reports `best_score_by_round`, `best_score_by_cost`, `rounds`, and `quality_at_budget` alongside the three components, and the objective is derived from those artifacts rather than kept as the only output.
- Every tree carries a compatibility fingerprint and an explicit `replayable` flag; each score states the pool partition it was computed over, and no revision is selected on stale-only evidence.
- Live rounds report replay fidelity — selection, batch size, stop decision, outcome availability, score, cost — and divergence past the configured tolerance pauses redeployment.
- Documentation, bilingual pairs, and the recorded-session snapshots required by the testing policy land in the same change.

## Risks

- **No score exists yet.** The whole mechanism is blocked on a task-owned quality signal. If the project invents a convenient proxy inside the harness, a self-improving loop will optimize the proxy and lock that in.
- **Node granularity is provisional in one respect.** This document records one attempt per node and one decision round per policy decision, because session-granularity nodes make replay cheap to build and nearly useless while attempt-granularity nodes are useful and require per-attempt workspace capture; what phase 1 still has to settle is whether a subagent run and a refinement round are comparable enough to share one cost unit, with recorded cost as the fallback.
- **Workspace capture costs disk.** One capture per attempt, content-addressed, is new storage growth the harness does not have today; retention prunes those artifacts only, and the tree skeleton and scores survive pruning (see D).
- **Realized trees are narrow.** Fan-out caps of 8 subagents and 10 parallel tool calls bound how much of the search space any recorded tree covers; widening them changes product behaviour and is not part of this proposal.
- **Monotone replay is not monotone live.** The guarantee is over fixed history only, so a selected revision can still regress a live run.
- **Model-written policy code is a security surface.** Executing and installing rewritten policy code needs the existing sandbox and an explicit review gate; auto-install into a shipped profile is out of scope.
- **Replay ceiling misread as a stopping signal.** A policy tuned to the recorded ceiling stops early on live runs where the ceiling is not the live optimum; the paper reports this as a live failure mode, not a theoretical one.
- **Lottery-ticket bias.** `max` aggregation rewards one lucky node over ninety-nine useless attempts, which is the paper's choice and not obviously this harness's; `qualityDefinition` exists so the bias is a decision, and phase 3 is where it is decided.
- **Staleness is the expected failure mode.** Trees stay structurally replayable while models, prompts, tools, and environments move, so a pool that ignores fingerprints will score revisions against history that no longer describes the deployment.
- **Evaluation gaming is a separate threat from sandbox escape.** A policy that cannot leave the sandbox can still manipulate the replay interface, request malformed actions, or exploit error handling; purity and the frozen observation are the contract that covers it, and neither replaces the other.
- **Decision rounds add recording surface.** Rounds, frontiers, and policy identity are new events on the hot path of every run; they must stay small enough that recording never becomes the reason a run is slow, and the baseline provider has to record them before any policy exists.

## Open questions

- How is a decision round delimited in a ralph-style loop versus subagent fan-out — per model step, per followup batch, or per explicit plan?
- Who declares the score: a task configuration, a tool the model calls, or an evaluator plugin the deployment mounts?
- Should the policy decide batching only across attempts, or also across tool calls inside one step, where `executeToolCalls` already runs a bounded pool?
- Do retry settlements (`assistant/attempt`, `llm/retry`) become nodes, or stay attributed to the attempt that contains them?
- Does the first implementation live in `packages/experimental/` until the seam is proven, and which shipped profile is the first to opt in?
- Should `maxGoalRounds` be exposed to the policy as part of its observed prefix, so a plan can size itself to the remaining budget?
- Should a subagent run and a refinement round be priced by one cost unit, or by their recorded token and time cost?
- Which fingerprint components are cheap enough to hash on every run, and which belong to deployment configuration the harness cannot see?
- Does the fidelity metric belong in the harness, or in the improvement loop's own tooling?
