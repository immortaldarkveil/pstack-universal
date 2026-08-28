---
name: pstack-universal
description: "Universal reverse-engineering of cursor/plugins pstack (poteto-mode) into an agent-agnostic skill. For any agent, any env. Dispatcher + 21 principles + playbooks + leaf skills. Use when user gives a goal and a checkable done condition — the skill picks the playbook, sequences work into verifiable units, and proves it works on the real artifact."
---

# pstack-universal — any-agent skill

> **Reverse-engineered from** `cursor/plugins` / `pstack` (`poteto-mode`) — 157 files, 7800+ lines, 22 playbooks, 21 principles, ~40 leaf skills.
> Source fork: `https://github.com/immortaldarkveil/plugins` (from `cursor/plugins` @ main).
> This one file replaces the plugin. No Cursor, no MCP, no model slugs required. Any agent that can read markdown, run shell commands, and delegate to subagents can follow it.
>
> **Rule for the agent reading this:** if the task is multi-step (investigation, bug, feature, refactor, perf, migration, overnight run, or "figure it out"), your first todo is to read the Principles section below in full. No exceptions.

## 1. What this is

pstack works when you stop micromanaging the agent. You describe **what you want** and **how you'll know it's done**. The agent (dispatcher) picks the playbook, copies its steps into a todo list, and calls other skills as each step needs them. You don't name playbooks or list skills. A good prompt is:

```
the export writes duplicate rows when a retry lands mid-run. repro first, then fix and verify.
```

"repro first" and a checkable outcome are all the routing signal the dispatcher needs. It matches Bug fix, copies steps into todos, and calls the right skills as each step fires.

```
Your prompt → dispatcher → read Principles → match playbook → copy steps verbatim into todos → run skills per step → verify on real artifact → report with evidence
```

Sticky: once the mode is on, it stays on for the conversation until you say "new task" or "stop."

## 2. How to invoke (human side)

Say the goal in your own words plus the check. Don't enumerate skills.

**Good prompts by intent:**
- Bug: `users get two notifications after a retry. repro first, then fix and verify.`
- Feature: `add a --json flag. text output stays byte-identical. verify both.`
- Refactor: `move parsing into one module, zero behavior change. record output before and prove unchanged after.`
- Perf: `startup takes 1.8s on this fixture. trace it, fix measured cause, show before/after.`
- Research: `how does the sync service decide ownership? don't change code yet.`
- Design: `design the import pipeline before writing code. i care most about caller usage.`
- Long run: `im stepping away. keep going until migration check reports zero old callers. log decisions. don't ask before committing.`
- New task mid-chat: `new task. figure out why cache survives logout. don't change code yet.`

**Anti-prompt:** `use /how then /architect then /arena` — the playbook already sequences them. Name a skill only to override.

**Parallel work:** `branch off <base> in a fresh worktree, then do X` — isolates agents so they don't stomp files. Prune later when disk is tight.

## 3. Dispatcher — how the agent routes

1. Read Principles section in full. Name each principle that shaped a decision and what it changed. A citation with no decision is a miss.
2. Match the task to a playbook (table below). If no narrower playbook fits, or it's large/cross-cutting, or the user will review later → `figure-it-out`.
3. Copy that playbook's steps **verbatim** into your todo list as the first todos, before any task-specific todos. A step you skip stays with `skip: <reason>` — never silently drop.
4. Run each step. Call leaf skills when the step says so.
5. Verify on the real artifact per `prove-it-works`. `inconclusive` is not `pass`.
6. Write reply per `Writing the reply`. Name who benefits and what a maintainer inherits. Paste evidence verbatim.

### Playbook router

| User signal | Playbook |
|---|---|
| Read-only: how does X work, why was Y built this way, are we sure about Z, should we do X or Y | Investigation |
| Defect to reproduce and fix | Bug fix |
| Measured slowness to trace | Perf issue |
| One metric to improve iteratively to a target | Hillclimb |
| Leak, idle spin, glitch from live instrumentation (deliverable is diagnosis) | Runtime forensics |
| Diagnosis from a captured profile/trace/snapshot | Trace forensics |
| New or changed behavior | Feature |
| Behavior-preserving structure change (rename, extract, move, dedupe) | Refactoring |
| Throwaway sketch to settle a design/behavioral choice cheaply | Prototype |
| Pixel-exact equivalence between two implementations | Visual parity |
| Testing how a skill/prompt change affects behavior | Eval |
| Work spans phases or stacked PRs, deliverable is a plan not code | Multi-phase plan |
| Long/unattended task driven to a predicate ("run until X") | Autonomous run |
| Project-scale, multi-day, many stacked PRs, dozens of agents | Orchestrate |
| Drive a PR/stack to merge-ready (conflicts, reviews, CI) | Babysit |
| Verify a green stack and land the contiguous verified run | Shipping |
| Take over prior work from a branch/transcript | Session pickup |
| Suspend cleanly to resume later | Pause safely |
| Clean up worktrees/simulators | Worktree cleanup |
| Writing/editing a SKILL.md | Authoring a skill |
| Queue of independent PRs to merged / stacked | Autopilot-full / Autopilot-stack |

Full paths in original: `pstack/skills/poteto-mode/playbooks/<name>.md`. This file distills each below.

## 4. Non-negotiables (every multi-step task)

- **Todo first.** Start every multi-step task with a todolist whose first item is to read the Principles section in full.
- **Name the data shape first.** Any code → choose its organizing structure per `model-the-domain` before writing logic.
- **Boundary-crossing code → architect.** Parallel design exploration before implementing.
- **Parallel fan-out → swarm** for coverage/races/partitions; **arena** for design/code bakeoffs with base+graft.
- **Contested design → interrogate** (multi-model adversarial) before shipping.
- **Nontrivial multi-step → throughput checkpoint** (Feature step 3 — four items, keep with `n/a: <reason>` if genuinely inapplicable).
- **Any prose surface → unslop.** Any PR/docs/commit → technical-writing + unslop. Before commit → deslop + no-comments.
- **Any PR-status request → Babysit playbook** — not a generic babysit. Includes "babysit this", "get it green", "check on PR X".
- **Asked to land a green stack → Shipping playbook.** Green is not safe. Independent per-PR verdict required.
- **Long/autonomous/multi-phase or "going to bed" → decision trail** via show-me-your-work. Commit trail when stakes need audit.
- **Code change → prove-it-works on the real artifact.** Not "it compiles." Repro, then fix, then re-run on same surface.
- **If you ask a question** about which approach / what should this do → classify first. If answer is observable by running something (behavior, timing, layout, perf, eval), prototype it — don't ask the human. Reserve questions for genuine product/preference calls no experiment can settle.

## 5. Principles — 21 rules, each with when, rule, test

Read the leaf file in full before applying. You must name the decision each changed.

### Core (how much to build, when to rethink)

**Laziness Protocol** — refactoring, sizing diff, tempted to add abstractions/layers.
- Bias to deletion and smallest change that solves it. Prefer deletion, flat hierarchy, consolidate decisions, minimize diff. Small leaks compound.
- Test: would a human maintainer find this exhausting?

**Foundational Thinking** — before writing logic; choosing types, scaffold vs feature, what concurrent actors share.
- Data structures first — right shape makes logic obvious. Scaffold first (CI, types, test infra) if every later phase benefits. Scaffold before features, tests before fixes. Small single-purpose commits.
- Test: does every subsequent phase benefit from this existing?

**Redesign from First Principles** — integrating a new requirement into existing design.
- Redesign as if requirement had been foundational day one. Don't bolt on. Read all affected files, propagate through types/docs/examples, deliver incrementally.
- Test: would a fresh design have looked like this?

**Subtract Before You Add** — sequencing addition/refactor/rewrite.
- Remove dead weight first, then build on simpler base. Cut before polish. No speculative validators/parsers. Delete stubs with no novel content.
- Test: did you cut before you built?

**Minimize Reader Load** — reviewing/shaping code hard to trace.
- Two axes: layers to trace, state to hold. Collapse one-caller wrappers, pass-through layers, shrink mutable scope (pure > locals > fields > module > globals). Interface must compress, not repeat.
- Test: can a new reader answer "where does X come from?" in 30s?

**Outcome-Oriented Execution** — planned rewrites/migrations with phase boundaries.
- Converge on target architecture. Don't preserve throwaway compat. Intermediate breakage is OK when planned/scoped/reversible. Verify at boundaries.
- Test: are you optimizing the end state, not the intermediate?

**Experience First** — product/UX/feature-scope tradeoffs.
- Choose user delight over impl convenience. Ship fewer polished features over many rough ones. Prototype before committing. Sweat details.
- Test: what does the consumer (end user or importing colleague) actually notice?

**Exhaust the Design Space** — novel interaction/architecture with no precedent.
- Build 2-3 competing prototypes, compare side by side before committing. Second flavor of same shape doesn't count.
- Test: did you compare real alternatives, not variants?

**Build the Lever** — any non-trivial work.
- Build the tool that does/proves it (codemod, script, generator, delegate skill) not by hand. Tool is the artifact reviewer reruns. Do one unit by hand to learn recipe, then build lever, diff against hand-done, make rerunnable. Deterministic lever beats fan-out.
- Test: if you cited this, is there a script/codemod/generator/skill in the diff? If not, you didn't apply it.

### Architecture (where state/validation/compat live)

**Model the Domain** — stateful logic, branching or repeated shape assumptions across files.
- Encode domain in structure (state machine, typed model, table/registry, reducer, boundary, queue/index/graph) not scattered conditionals. Execution order is not ownership.
- Test: did a new feature grow an if/else chain by one more branch? You skipped this.

**Boundary Discipline** — validation, error handling, framework adapters.
- Guards at system boundaries (CLI, config, network, external APIs). Trust internal types. Business logic in pure functions. Don't re-export transport/wire types.
- Test: is this data crossing a boundary right now? If not, validation is redundant.

**Type System Discipline** — designing types/signatures in typed languages.
- Make illegal states unrepresentable (sum types). Brand primitives (UserId != OrderId). Parse external data at boundaries. Don't lie with casts. Exhaustive matching must fail compilation on new variant. Derive from authoritative schema.
- Test: can you write a comment explaining when field combination is valid? Type is too loose.

**Make Operations Idempotent** — commands/lifecycle loops amid crashes/retries.
- Converge to same end state regardless of runs. Scan existing state, clean stale artifacts, PID-based stale locks. Ask: twice in a row? Crash at any point?
- Test: does re-execution converge?

**Migrate Callers Then Delete Legacy APIs** — new internal API while old callers exist.
- Migrate all callers and delete old API in same wave. No compat layer. Inventory → migrate → delete. Treat adapters as exceptional/time-boxed.
- Test: are both old and new APIs still alive?

**Separate Before Serializing Shared State** — concurrent actors might write same file/branch/key/object.
- Default: eliminate sharing. Give each actor its own file/branch/key, merge at read boundary. Only when one shared writer is real invariant → serialize structurally (lockfile, sequential phase, single-writer). Instructions are not concurrency control.
- Test: do two workers write one `state.json`? Split to `indexer-state.json` + `metrics-state.json`.

### Verification (what counts as proof)

**Prove It Works** — after task, before done.
- Verify against real artifact, not proxy/self-report/"it compiles." Check process liveness directly, read actual value, test full chain input→output, full comm path for integrations. Script the check deterministically so reviewer can rerun. Inspect artifacts, not delegate summaries.
- Steps: Build (necessary, not sufficient) → run and exercise real path → check full chain → capture evidence.
- Test: how do you prove this actually works, with a command a reviewer can rerun?

**Fix Root Causes** — debugging.
- Reproduce first — can't prove fix without repro. Ask why until root. Resist nil-check guards that silence crashes. Check pattern not just instance (grep). When stuck, instrument. State before code for "fails after restart" (clear state file test).
- Test: does explanation need a paragraph to justify workaround? Fix the code.

**Sequence Work into Verifiable Units** — multi-step work (sweeps, migrations, repeats) and stacking commits/PRs.
- Break into small units each ending in a check, verify before next, order delivery so sequence proves itself. Canonical: failing test first (red), then fix (green). Rebase onto clean trunk so checks measure real baseline. Don't batch edits and verify once at end.
- Test: can reviewer replay commits and watch red→green?

### Delegation (keeping parallel work sane)

**Guard the Context Window** — context fills up (large outputs, long files, repeated reads, fan-out).
- Route bulk to subagents, keep summaries in main thread. Don't read what you won't use. Size phases, cap scope.
- Test: is main thread holding raw payloads that a subagent could summarize?

**Never Block on the Human** — tempted to ask "should I do X?" on reversible work.
- Proceed, present result, let human course-correct. Reserve questions for genuine ambiguity. Code is cheap, attention is scarce.
- Boundaries: irreversible (force-push, delete prod data, external messages) → pause. Reversible (write code, edit notes) → proceed.
- Test: is this reversible? Then don't ask.

### Meta

**Encode Lessons in Structure** — catch yourself writing same instruction second time, or recurring correction.
- Encode as lint, metadata flag, runtime check, or script — not more text. Pick strongest rung that works (unrepresentable state > lint/banned API > canonical helper > runtime check). Capture every correction, route to right layer, close loop.
- Test: did you encode or just acknowledge?

## 6. Playbooks — distilled steps (copy verbatim into todos)

### Investigation (read-only question)
1. Route through `how` (Explain for narrow, Critique for "are we sure?"). For motivation, also route through `why`.
2. Throughput checkpoint: `n/a, read-only investigation`.
3. Produce `how`-shaped output (Overview / Key Concepts / How It Works / Where Things Live / Gotchas) or recommendation with tradeoffs table.
4. Apply `unslop` to reply.
- No PR, no babysit, no architect unless investigation precedes code change.
- Reply: investigation output with pushback if premise is wrong.

### Bug fix
1. Reproduce yourself on matching surface (browser/CDP, PTY/TUI, curl/service). Don't hand repro to user. If it won't repro, synthesize trigger, tighten conditions, instrument until it fires. Can't repro = can't prove fixed.
2. Binary-search the cause. Form hypotheses, seed with `how` + `why`, cut most space each pass, get runtime evidence, eliminate. When stuck, instrument and read live. Use loop mechanism for long hunts. Confirm surviving mechanism with runtime evidence before step 3.
3. Plan fix. If crosses function boundary → `architect` first. Delegate implementation to subagent with specific scope; review diff.
4. Verify on same surface; original repro now passes. "Inconclusive" or wrong surface = not a pass. Unit tests show branch behavior, not bug absence.
5. Stage commits so failing repro lands before fix (sequence-verifiable-units). See `tdd` for failing-test-first when cheap; skip when expensive/integration-heavy/unclear.
6. Opening a PR.
- Reply: what was broken, root cause, fix, verification — paste failing→passing output verbatim.

### Feature
1. `how` over affected subsystem.
2. `architect` for parallel design exploration. `skip: <reason>` if skipped — don't fold silently.
3. Write throughput checkpoint (4 todos):
   - Blocking first steps (gates before fan-out)
   - Independent workstreams (disjoint files/services/layers parallelize; shared writes serialize)
   - Shared mutable state (default split target per separate-before-serializing; serialize only for real invariants)
   - Smallest safe decomposition (if one worker best, name why)
4. Delegate code to subagent with specific scope (file paths, named data shape + organizing structure per model-the-domain, success criteria); review diff. If multiple valid shapes → delegate via `arena` (mandatory — no skip, Laziness Protocol doesn't override). No narrating comments. Surgical edits, re-ground for upstream-derived files. Port shared-primitive improvements to all consumers, verify each. Commit liberally.
5. Verify on matching surface. Flag inconclusive.
6. Rebase into small ordered commits; stack follow-ups per sequence-verifiable-units.
7. If contested → `interrogate` before shipping.
8. Opening a PR.
- Reply: what you built, choices + why, open decisions, design-alternatives table.

### Refactoring (behavior-preserving)
1. Pin behavior first — capture current output/behavior as baseline before moving structure.
2. `how` over affected area; `architect` if crosses boundary.
3. Throughput checkpoint as in Feature.
4. Delegate via subagent with shape (state machine / table / typed model) + invariant that behavior is unchanged. No narrating comments. Apply laziness protocol, minimize reader load.
5. Verify behavior unchanged (replay baseline, diff output, run targeted tests). Any visible change → not a refactor, re-route.
6. Sequence-verifiable-units commits (one structure move per commit).
7. Opening a PR.
- Reply: what moved where, why safe, proof of unchanged behavior.

### Prototype (throwaway sketch to decide)
1. State the empirical question only an experiment can answer (layout, timing, behavior, API viability). If deliverable is read-only cited answer → Investigation, not Prototype.
2. If answer is fact observable by running → don't ask human. Sketch it.
3. Build throwaway in isolated branch/worktree. Cheap, no polish. One or two shapes if design space is open.
4. Observe result, capture evidence (screenshot, log, trace).
5. Let result decide — no human decision needed. Keep branch+SHA+screenshots for Appendix if this feeds a plan.
6. Discard or promote deliberately — don't silently merge throwaway.

### Perf issue (one-off measured slowness)
1. Capture baseline trace via real surface harness.
2. `how` to ground hypotheses; don't claim ceiling without running it.
3. Generate hypotheses from 8 families — only when trace shows its signal:
   - Elimination (delete work nobody consumes), Divide&conquer (chunk/shard/parallelize), Caching (reuse same input), Indirection (index/queue/handle), Batching (coalesce per-op overhead), others per trace. Focused fix for dominant cost beats applying all 8.
4. Fix dominant cause, measure before/after with frozen harness, gates green.
5. Verify on real surface with numbers.
6. Opening a PR.
- Reply: baseline → final with delta, cause, fix, measurement.

### Hillclimb (sustained improvement of one metric to target)
1. Ground workload+architecture via `how`; name realistic workload dimensions; pick case that reproduces complaint. Fix metric, direction, stop predicate (target + floor on attempts, e.g. "50% better and ≥10 iterations").
2. Build frozen measurement harness, prove sensitivity (contrasting workloads separate), baseline + green gate before changes. One command emits metric (median of N). Changing harness invalidates earlier numbers.
3. Open `decisions.tsv` per show-me-your-work: id, hypothesis, change, before, after, delta, tests, verdict (kept/reverted), note.
4. Ground each hypothesis in architecture model (specific mechanism, not "try memo").
5. Loop one hypothesis per iteration: subagent does tight-scope change → measure before/after with frozen harness + gate → keep only if metric past noise and gate green else revert fully → one commit per kept fix (`git add <files>` only) → log row either way. Never stack untested changes.
6. Push past first plateau — pivot category, combine near-misses, re-read source, try radical before concluding.
7. Stop when predicate met or remaining ideas marginal and not worth cost. Don't relax predicate.
8. Opening a PR with commits in order landed.
- Reply: metric, target, baseline→final %, iterations kept vs reverted, each kept fix one line, decisions.tsv path, next best idea.

### Runtime forensics / Trace forensics (diagnosis, not fix)
- Ground with `how` + instrumentation or captured artifact (cpuprofile, trace, heap snapshot).
- Deliverable is diagnosis with evidence and mechanism, not code change. Route fix through Bug fix / Perf after.

### Visual parity
1. Capture reference screenshots of old version before changing anything (baseline).
2. Build harness that captures new version same way.
3. Pixel-diff 0 is the proof. Iterate until diff 0 or delta is intentional and documented.
4. Keep reference set for reviewer rerun.

### Eval (testing a skill/prompt change)
1. Frame variant + success rubric (3-6 concrete criteria) — judge only, never candidate.
2. Sanitized environments per candidate (project-shaped dir names, no `eval`/`candidate`/`arena` words, no chain cues, no prompt leakage). Candidate prompt looks like organic user request.
3. One organic prompt, same to each candidate, N parallel candidates on different models/subagents, each in own dir.
4. One blinded judge (different model family) sees outputs by sanitized label only, scores against rubric on one scale in single pass.
5. Verify chain from transcripts (which files actually read) not self-report. Read every output yourself, compare to judge, synthesize.
6. Reply: variant, rubric, per-candidate notes, judge verdict, your synthesis, promote yes/no.

### Multi-phase / multi-PR plan (deliverable is plan, not code)
- If 1-2 files with obvious approach → skip plan, say so.
- Prototype open questions before writing; keep branch/SHA/screenshots for Appendix A.
- Explore via subagents (general-purpose), each returns file pointers, conventions, test commands, entry points.
- Copy skeleton (How to read, Program checklist, Per-PR Build/You see/Verify unit+live+perf/Review gate/Merge, Appendices A-D), fill every placeholder. Write under docs/ unless told otherwise. One PR = one change with own evidence per sequence-verifiable-units.
- Verification rule every block: "Tests alone are not sufficient. A PR is verified only when its unit, live, and perf boxes are all checked." Live = ten lanes driving real surface via harness; perf = metric, probe, trunk baseline first, rule with failing number.
- Apply technical-writing + unslop; prose is how-to. Appendices hold explanation/reference. Enforce with `check-plan` script (skeleton shape, verification rule, punctuation).
- Hand back plan path + check output, stop — execution starts on explicit go via Autopilot-full / Autopilot-stack / Orchestrate (pick per rule).

### Autonomous run (one task to predicate, unattended)
- Design phases via figure-it-out + decision log wiring before code.
- Loop: check finish condition → smallest justified change → verify on real artifact → keep if advanced else revert → commit if kept → one log row. Plateau → pivot, not stop. Don't relax predicate.
- Borrow only wake mechanism from platform (/loop or heartbeat), not its stop rule. Stop when predicate met or truly stuck with write-up.
- Reply: predicate, iterations, kept vs reverted, trail path, verification.

### Orchestrate (program that outlives one agent — use sparingly)
- Done predicate countable ("all 126 units merged, each ledger-verified"). Quantify scope, effort, stacks, wall-clock. If one agent could finish in budget → use Autonomous run instead (measured: orchestrate ceremony turned 30min 12-unit job into 1 unit).
- Install runtime: init store (`orchestrate/<slug>/` with preferences.md, overview.md, units.tsv, frontier.json, ledger.tsv, inbox/, decisions.tsv, status.md), standing orders pasted verbatim into every spawn/resume.
- Pilot one unit end-to-end (brief → worker → verify → ledger → merge), fix contract from evidence, then rolling window of workers up to ~10 in-flight, refill as children finish (never blocking batches). Sub-coordinator only past one-drain threshold.
- Drain queue at batch points (end of critical section, rollup, watcher wake, before human report): inbox push → drain → classify → update tables → status → spawn next wave.
- Stack safety: frontier recomputed from authoritative source after every merge; exactly one stacker runs stack ops; workers never rebase; PR closes/retargets via stacker only.
- Verification: ledger `live-ui-verified | unit-test-verified | type-check-only | verifier-blocked | verifier-failed` keyed by PR+SHA; new SHA voids verdict; behavioral work needs better than type-check-only.
- Liveness: never resume to check liveness; probe ledger/units/gh/branches; synthetic death row for silent deaths; retry by mode (cap→smaller scope, network→retry as-is, tool-error→different model); zombie reconciliation vs frontier/ledger.
- Escalation: human only for irreversible, genuine product calls, contradictory standing orders, dead ends; park in gates.md, route work around.
- Roles never write code on coordinator; landing is continuous from first verified unit; schedule landing by ~70% of budget; 30-min audit tick re-reads execution playbook + goal.

### Session pickup / Pause safely
- Pickup: treat prior trail as authoritative. Reconstruct branch state+decisions, name resume point, verify inherited claims vs original goal, don't redo finished work. Recompute frontier, reattach cloud work by PR/branch, respawn sub-coordinators from stored briefs.
- Pause: stop at safe boundary (finish atomic step or back out), don't cross irreversible line, commit `wip:` so nothing lost, write resume note off-context (intent, progress, verified, next steps, key files, gotchas) to `/tmp/<slug>-resume.md` or decisions.tsv pointer.

### Worktree cleanup
- Classify every worktree by merge state, uncommitted work, active chats; delete only evidence-cleared, pause for user on uncommitted.

### Opening a PR (invoked at end of every code playbook)
- Work from git worktree off main; subagents inherit it. Dirty branch → patch out, fresh worktree, apply.
- Commits: commit liberally during work; rebase into small ordered commits before opening. Each commit landable, story-ordered (failing test → fix, subtraction → reshape, baseline → treatment, scaffold → feature).
- Before commit: deslop pass (remove narrating comments, unsupported guards, dead compat, unrelated edits). Before review: no-comments pass (fresh reviewer). Titles: Conventional Commits `type(scope): subject` (feat, fix, docs, refactor, test, chore, perf), imperative, short, no period, name real symbol. Descriptions sections in order: `## Why` (intent + why this approach), `## Scope` (facts from diff, both sides of renames, in/out), `## Tradeoffs` (real choices only), `## Blast Radius` (who/what touched, safe/risky why, cost if not merged), `## Verification` (how each check ran, rigor, outcome, real paths like PTY/browser/tests, outcome not just command). After sections, attach evidence. No `## Summary`/`## Test plan` boilerplate.
- Open ready, never draft (`draft:false`; if draft → ready). Five narrow PRs beat one fat one; stack follow-ups; branch from main only for independent work; rebase on main before stack work.
- Opening does not start babysit — post URL, keep building, separate babysit pass only when user asks after whole stack exists. Subagent that opens PR runs interrogate + deslop + no-comments, returns URL, doesn't babysit.

### Babysit (drive PR/stack to merge-ready)
- Trigger: user asks "babysit", "get it green", "check on PR X" — answer status without starting loop if just checking. Declare mode before polling; playbook step 1 maps request to mode. Not triggered by merely opening PR.
- Order: conflicts → review threads → CI. Every known fix batches into one push so checks restart once.
- Comment triage skeptical — assess each on merits, dismiss noise with concrete reason. For bot security/bugbot comments, see bugbot-triage reference.
- Reaching for drive inside phase agent stops that agent finishing its turn — declare mode first.

### Shipping (land the stack)
- Green is not safe. Nothing armed before independent per-PR verdict, only contiguous verified run from root lands.
- One fresh verifier per PR (different model family from worker) proves behavior live at that PR's head; verifier never the worker. Swarm lanes: unit + live + perf + audit.
- Clean only when every lane PASS; findings back to owner; new head → fresh swarm + fresh verdict. Rebased onto current trunk after verdict with patch-id unchanged.
- Merge rule per execution playbook: owner squash-merges own PR, or root appends to stack and human lands.
- Babysit owns conflicts/reviews/CI per above; Shipping owns independent verification + landing. Don't conflate.

### Authoring a skill
- Route through skill generator's draft/test/iterate loop; validate frontmatter+links; ship via Opening a PR. Agent prose has higher bar than human prose — unhelpful sentence becomes instruction.

## 7. Leaf skills — when and how

### how — how does X work
- **When:** nontrivial change, architecture decision, "are we sure?" Use before editing what you don't understand.
- **How:** assess simple (single module, narrow) vs complex (subsystem, cross-cutting):
  - Simple → one explorer that does Glob/Grep/Read and writes explanation directly.
  - Complex → fan out 2-4 explorers each with distinct slice (data model, request path, config/metrics), read real code, return findings; then one explainer synthesizes. Explorers read-only, fast model. Explainer does the prose.
- **Output:** Overview (what it is, why exists), Key Concepts (types/abstractions), How It Works (trigger → steps → data flow → decisions, prose not pseudocode, file:line refs), Where Things Live (map), Gotchas (non-obvious, sharp edges).
- **Critique mode:** explain first, then spawn one critic per model family with rubric, each reads explanation + file paths, then lead sorts into Act on / Consider / Noted / Dismissed.

### why — why was it built this way
- **When:** motivation, tradeoffs, edge cases, thresholds, regressions, "why not Y?"
- **How:** anchor in concrete code first (blame, log --follow, gh pr view). Then enumerate evidence categories and query all available in parallel: source control (always), tracker (Linear/Jira/issues), docs (Notion/Confluence), chat (Slack), infra observability (Datadog), error tracking (Sentry), analytics warehouse (Databricks/Snowflake) — adapt to what's available. One investigator per category. Null results are findings. Synthesizer separates direct evidence vs inference vs gaps, with citations (commit hash, PR#, ticket ID, doc URL, permalink, file:line). Hedge: "appears to," "likely," "suggests" when indirect.
- **Output:** Question, Code in question, What we found (cited), What we can infer (chain), Competing hypotheses, What we don't know (explicit gaps), Sources consulted (one line per category: what searched, what found or skipped why). If why precedes a change → convert to Preserve/Change/Avoid/Risk constraints.

### teach — explain so a person actually understands
- **When:** "teach me X", "convince me this fixes cause not symptom"
- **How:** `how` + `why` in parallel (one may suffice for small), weave into one plain explanation. Lead with definition, tie to case, build diagram-by-diagram (each adds one part), offer to go deeper. No quizzes, no pacing theater. Plain spoken English via unslop.
- **Reply:** explanation itself, never a report.

### recall — where did I leave off
- **When:** "recall X", "catch me up", before resuming work.
- **How:** classify scope (time window default 7d, topic, workspace). Fan out across chat transcripts (ordered by mtime, grep topic, skip current/eval noise) + sweep shared record via `why` investigators when target names feature/file/subsystem. Verify PRs/branches/tickets live via git/gh. Write brief grouped by thread.
- **Output contract:** Capsule (≤5 bullets, what + where stands), Threads (one line each with tag: [merged #N] [open PR #N] [in flight <branch>] [verified, uncommitted] [reverted #N] [planned, not started]), Problems (≤5 recurring), Next move (single concrete action). Cites by transcript ID / PR# / ticket / permalink.

### architect — settle the shape before code
- **When:** code crosses function boundary, moves ownership/layers.
- **How:** ground with `how` (+ `why` if moving layers) → run `arena` with diverse isolated opinionated candidates whose usage is written first, then types/signatures/module map → read-only judge on different family scores vs rubric → pick base, graft best parts, verify synthesized result. With checkpoint → stop and show before implementing. Phase D deviations worth surfacing; repeated same-shape friction → scrap and redesign from first principles (re-run how, subtract-then-add, back to arena).
- **Output:** usage sketch, type sketch, module map, rationale per template, including alternatives considered and synthesis decision.

### swarm — fan out N parallel workers and aggregate
- **When:** coverage matrices, races, gauntlets, exploration partitions.
- **How:** Frame (done predicate + artifact, shape: partition / race / mix, declare selection rule `first pass` / `rank all` / `best-of`, set N, pick model), Fan out (one message, all N workers, each own writable output: worktree/branch or /tmp/swarm-<slug>/worker-<n>/, standalone brief with goal/scope/slice/verify/report), Aggregate (table, one-line evidenced issues, gaps/dropouts), Report (one consolidated report with table + issues + gaps + rule). Workers report PASS/ISSUES/BLOCKED with evidence. Dropouts noted, proceed with N-1.

### arena — parallel attempts, pick base, graft best
- **When:** design or code where one attempt would lock in wrong shape; standalone decision (naming, format, algorithm).
- **How:** Frame (artifact, rubric 3-6 concrete criteria, runners list, output paths isolated), Fan out (N subagents same prompt, each writes artifact + rationale naming alternatives rejected), Cross-judge (one judge on different family, sees candidates by label not model, scores vs rubric), Pick base (read every candidate end-to-end, criterion-by-criterion, compare to judge — agreement confirms, disagreement means bias/ambiguous rubric), Graft (walk losers, port 1-2 things per candidate by hand, redesign-not-paste, record grafts + rejections), Verify (prove-it-works on synthesized artifact — arena doesn't earn a pass).
- **Output:** synthesized artifact + synthesis note (base, grafts with source, rejections why, dropouts, verification). N converging → ship consensus; N wildly diverging → reframe and rerun.

### interrogate — adversarial multi-model review
- **When:** "interrogate", "stress test", "find blind spots", contested design before shipping.
- **How:** Determine scope (pointed files or diff on branch vs main), state intent paragraph, spawn one reviewer per model family with same prompt+rubric+code-quality lens (read-only), each produces structured findings, lead synthesizes:
  - Consensus (2+ models) = highest signal; lone-model = lower confidence; deduplicate; note disagreements.
  - Lead judgment categories: Act on (real correctness/security/maintainability, would block), Consider (legitimate but cost unclear), Noted (valid low-priority), Dismissed (wrong/nitpick/missing context) — each with models + one-line rationale + agreement map.
- **Output:** Intent, Reviewers (model, N findings), Act on / Consider / Noted / Dismissed, Agreement map. Apply nothing automatically.

### tdd — failing test first (when cheap)
- **When:** user asks for TDD, or bug has obvious cheap local test target. Skip when test path unclear/expensive/integration-heavy/production-only.
- **How:** understand intended vs current behavior + smallest observable repro → choose narrowest existing test path → write smallest focused failing test encoding intended behavior → run it (must fail for intended reason) → smallest production fix → rerun (must pass) → run nearby validation (adjacent tests, typecheck, lint).
- **If impractical:** say why explicitly, use closest executable check (script, automation, snapshot, log assertion) instead of bad test. Prefer no test over bad test (mock-heavy, brittle, timing-dependent, expensive). Report failing-before evidence and passing-after evidence.

### blast-radius — what does this small change break elsewhere
- **When:** "blast radius", "what could this break" on a diff you don't trust.
- **How:** read diff + what it does differently (including unsaid part) → find one fact it's safe because of ("this only drops dead cache entries") → look where grep stops (library source + pinned version, microtasks/teardown, cross-language wire formats, flags, downstream) → rate each risk by likelihood × cost, cite file:line → prove the one fact by running code (import real lib, call exact function) — ladder: said so < line ref < walked failure < ran script < reproduced in app. Mark unproven if can't get to "ran it."
- **Output:** What it does, One safety fact + proof level + paste, Risks (real only, each with how breaks + likelihood/severity + check), Cleared, Before-you-merge cheapest catch (including script). For broad changes run as arena.

### unslop — cut AI tells, add human voice
- **When:** any prose surface — reply, docs, RFC, README, PR description, commit message.
- **How:** scan for 31 patterns (puffery, name-dropping, superficial -ing, promotional, vague attributions, formulaic challenges, AI vocab, fancy "is", "not just X but Y", rule of three, synonym cycling, false ranges, em dash/colon overuse, bold overuse, inline-header lists, title case headings, decorative emojis, curly quotes, sycophantic phrases, filler/hedging, generic conclusions, metaphor nouns, plain-speech fixes) → rewrite preserving meaning + tone → add soul (opinions, varied rhythm, acknowledge complexity, use "I" when fits, let mess in, be specific) → self-audit "what makes this obviously AI?"
- **Rules that matter most for code agents:** no em dash, colon before list OK but not mid-sentence connector, short declarative sentences ending in period, bold headers as own sentences, no inline-header lists, sentence case headings, straight quotes, active voice, plain words, one thought per sentence.

### no-comments — strip narrating comments via fresh reviewer
- **When:** before review, on diff or pointed files (default diff vs main including worktree).
- **How:** spawn fresh reviewer ("Comment Sicko") read-only to flag comments; inspect report — reject application-code edits, scope escapes, exception-protected deletions, misstated reasons, keep survives only with proof it's about something unchangeable; fix trivial flags by deleting dead path/dropping param/using real API else run `architect` once for set; implement smallest root-cause fix in scope removing every named workaround; constraint comments ("do not remove", "talk to X") → offer cheapest type/runtime/test/lint encoding, wait for approval if interactive else delete and report constraint open; report deletion count, restores, reruns, fixes, encodings, open work.
- Keep list: license headers, doc comments on public API, links that explain what code can't, behavior forced by external dependency you can't reshape. Everything else goes. Surprise in own code = refactor flag, not keep.

### figure-it-out — design the workflow when no playbook fits
- **When:** large migration, ambitious multi-part change, no narrower playbook, or work reviewed after stepping away (overnight contract). Scales rigor biased high.
- **How:** Frame (falsifiable done predicate per prove-it-works, quantified scope + blockers surfaced, rigor level), Design workflow (decompose into atomic landable units riskiest-first, harness before work with baseline, architect via arena for one-way doors, decide fan-out with isolation, write phase list for human review) → put design into todos → Run loop (hypothesis → smallest change → measure on real artifact → keep if advanced else revert → verify via artifacts not self-report → VERIFIED/NOT VERIFIED/INCONCLUSIVE, inconclusive ≠ pass; delegated work paired with judge) → Keep audit trail via show-me-your-work (row per unit, evidence as links, prefer committed scripts) → Verify whole against predicate + encode recurring corrections as gate/lint/script.
- **Reply:** playbook designed, rigor why, trail path, what's verified, what's open.

### show-me-your-work — decision trail for auditable runs
- **When:** long-running/unattended/multi-phase work; human reviews after fact.
- **How:** one canonical TSV, one row per decision/checkpoint (not every action). Columns: `ts` (ISO8601), `phase`, `decision`, `why` (plain words, principle plain not jargon), `evidence` (commit SHA / PR# / file:line / artifact path — pointer not prose), `result` (tests green / reverted / pixel-diff 0 / INCONCLUSIVE / open). Helper: `scripts/log.sh <logfile> <phase> <decision> <why> <evidence> <result>` (stamps ts, header on first use, escapes leading =+-@ for spreadsheets). Default local (`decisions.tsv` or `.audit/<slug>.tsv`, gitignored); commit only when reviewer needs trail (large port/migration). Append-only — wrong call gets new superseding row, never edit history. Audit log against transcript before handing back (every row maps to real action, evidence resolves, gaps added, padding dropped). Spawn cross-model reviewer to flag weak evidence / skipped verification / risky choices; reply ends with `reviewed by <model>` + Attention flags. `column -s$'\t' -t decisions.tsv` to view.

### create-verification-skill — make the app drivable (any stack)
- **When:** project has no scripted way to drive real app and prove behavior.
- **How:** interview repo not user — surface (web/CLI/TUI/API), run (dev command, ports, env, seed, auth), drive (existing harness first — Playwright/Cypress/expect/PTY/curl, then generic: browser/CDP for web/Electron, tmux/PTY for CLI/TUI, HTTP for services), observe (screenshots, transcripts, bodies, logs, exit, DB), isolate (can two instances run side by side?). If checkout broken → fix first. Generate `.<agent>/skills/verify-<app>/SKILL.md` with frontmatter + Launch (exact start + ready signal + teardown), Doctor (worth-driving check), Drive (real selectors/handles not coords), Evidence (real user path, capture action+resulting state + side effects, proof standards), Cleanup (kill what you started, evidence survives), Helpers (executable, invocation shown). Seed `features/README.md` + one file per top 3-5 features with H2s: Sub-features, How to get to it (user POV), Driving it with <harness>, Gotchas. Proof run: launch → doctor → drive ONE mapped feature → capture evidence → cleanup confirms evidence still exists. Offer maintenance loop.

### maintain-verification-skill — keep the harness honest
- Spawn one read-only reader per feature in parallel to audit map vs source, then one live pass driving every mapped feature. Outcome exactly one: `clean` (full coverage), `changed` (one PR of proven corrections confined to skill dir), `blocked` (reason). Never edits product code; product regression found live → report it.

### other leaf skills (distilled)
- **reflect:** after 5+ tool-call task that landed or hit dead ends, mine transcript + route learnings into skill edits via three reviewers (judgment/tooling/divergent) → synthesized Accepted/Rejected/Backlog → wait for explicit approval → trivial edit direct, substantive via skill generator, new skill via generator. Present full output before applying.
- **setup-pstack:** detect available models, load current config, map-confirm via question, validate slugs, write always-apply rule (one line per role; panel roles are lists where length = fan-out; `inherit`/`auto` means inherit parent model; delete line to restore default), confirm, optionally offer verification skill.
- **technical-writing:** pick Diátaxis mode (tutorial/how-to/reference/explanation), then sentence-by-sentence rule: who does what, one thought per sentence, nothing readable two ways. For docs/RFCs/PRs/commits.
- **typescript-best-practices:** loads when touching .ts/.tsx — discriminated unions, unknown at boundaries, exhaustive variants, schema-derived types.
- **automate-me:** mine recent transcripts for repeated preferences (replies, delegation, verification, code, prose, process), ask which are really you, draft `<your-name>-mode/SKILL.md` via skill generator + unslop, PR from worktree.
- **bro:** restate last message like human to human, no jargon, shorter — when reply is thorough but unclear.

## 8. Verification & shipping — the proof contract

**Prove it works** checklist before you say done:
- [ ] Built (necessary not sufficient)
- [ ] Ran real path on real artifact (CLI runs command, UI walks flow in running app, parser replays input, storage reads back value, perf compares profiles)
- [ ] Checked full chain input→output, including side effects (files, rows, messages)
- [ ] For integrations, tested full comm path end-to-end
- [ ] For delegated work, inspected diff/file/runtime — not delegate's summary
- [ ] Evidence captured to rerunnable script or step + artifact path pasted in reply
- [ ] Script comparison preferred over eyeball (old vs new compiled output diff)
- [ ] Trail committed only for large/ambitious work needing audit

**Sequence verifiable units:**
- Execute: verify each small unit before next, never batch-verify at end. One change → run check → proceed. Rebase on clean trunk first.
- Deliver: order commits so sequence proves itself to reviewer (red failing test → green fix; subtraction → reshape; baseline → treatment; scaffold → feature). Each commit landable alone.

**Opening a PR, shipping distilled in Playbooks section above** — keep PRs small, five narrow beats one fat, stack with branch tools, prove each PR independently before landing contiguous run.

## 9. Delegation — context, autonomy, parallelism

**Guard the context window:** isolate large payloads to subagents; main thread keeps summaries + file pointers (e.g. `file:line`), not raw dumps. Don't read what you won't use. Keep frequently used templates inline. Size phases, cap files per phase.

**Subagent defaults:** `run_in_background: true` where supported; isolate writable state (worktree/branch or `/tmp/<slug>/worker-<n>/`); pass explicit scope + verify contract; own every subagent's work — review diff and write your own summary, don't pass through. Prefer fresh spawn with consolidated scope over interrupt-chained resume (directives drop). Second opinion = same prompt vs different model; agreement is high-signal.

**Never block on human:** just do it for reversible work; use available tools; proceed, present result; let human course-correct async. Pause only for irreversible (force-push to shared branch, deploy, delete data, external message). Session overrides ("don't stop", "going to bed", "run until done") → keep going. `no` is acceptable answer — recommend with judgment, don't validate.

**Separate before serializing:** eliminate shared write target before adding locks/sequencing.

## 10. Writing the reply

Write clean as you draft — cleanup-afterwards fails. Rules:
- Short declarative sentences. One thought per sentence, ended with period.
- Long-dash character banned outright. File-list bullet joins filename to description as sentence: "`main.js` owns persistence." Header + text as own sentence.
- Colon before list is fine. Colon as mid-sentence connector is out — rewrite without it.
- Terse ≠ drop content. Every playbook-named section stays (details, tradeoffs, choices, open decisions).
- Frame impact for consumer and maintainer first — who benefits and what changes for them, then what next engineer inherits. If you can't say, explanation is off.
- Never fabricate link/citation/transcript ref. Link only artifacts you produced or read this session.
- Every playbook ends with reply in this style, PR link as `https://github.com/<owner>/<repo>/pull/<number>`.

Comments follow same rule — write clean as you go. No narrating phases like `// Phase 1: add cards`. Assertion/log string is the only doc needed: `assert(ok, 'persisted across restart')`. Keep comment only for non-obvious why code can't show.

## 11. Pitfalls — mistakes everyone makes once

- Enumerating skills in prompt. State goal+constraints. Name skill only to override.
- Vague finish condition. "Make it better" gives loop nothing to check. Give command/artifact predicate.
- Parallel agents in one worktree. Own worktree per attempt.
- Arena for coverage. Arena = one brief repeated + base/graft. Swarm = slices/races + aggregate.
- Accepting every review comment. Triage skeptically; dismiss noise with disproof on thread.
- Treating `auto` / `inherit-parent` as model slug. It means omit model → inherit parent.
- Reporting success off green build. Build proves compiles. Show real command/flow/value/profile + evidence.
- Writing SKILL.md freehand. Route through authoring playbook so validation happens.
- Editing skill mid-task. Fix in own PR, don't block.
- Duration as finish condition. "4 hours" is motion, not result.

## 12. Quick recipes (copy, swap in your paths/predicates)

```
# Understand before changing
how does this initialization work? then why did it break recently?

# Second opinion on design
arena this thread and our approach — compare proposals

# Parallel package checks
swarm check every package under packages/ against its check.sh. one worker per package. one report.

# Skeptical review (read-only)
interrogate the whole branch, but skeptically. don't change anything yet. no nitpicks unless actual bug/regression.

# Bug via failing test
repro the duplicate write first. if there's a cheap test path, tdd it. then fix and rerun.

# Overnight
im going to bed, keep going until every fixture passes. do not stop. keep a decision log i can audit.

# Redirect drift
i said the goal is to repro. i did not ask for a fix yet.
apply prove it works. show me the real output, not the build log.
unslop that, no emdashes

# Plain words
bro
```

## 13. File map in the original (for deep dives)

```
pstack/
  agents/ poteto-agent.md, comment-sicko.md
  docs/guide/ 01-setup … 10-recipes + README
  skills/
    poteto-mode/ SKILL.md + playbooks/*.md (22) + references/bugbot-triage.md + scripts/{bootstrap.ts, check-plan.mjs, orch/*, watch-pr/*, worktree-audit.sh}
    how/ why/ architect/ arena/ swarm/ interrogate/ tdd/ blast-radius/
    unslop/ no-comments/ reflect/ figure-it-out/ teach/ recall/
    create-verification-skill/ maintain-verification-skill/ setup-pstack/
    show-me-your-work/ technical-writing/ typescript-best-practices/
    principle-*/ (21)
  automations/benny/ skills/{reproduce-and-fix-issues, setup-benny, triage-issue-reports}
```

Leaf `SKILL.md` frontmatter always: `name`, `description`, `disable-model-invocation: true` where not user-invocable. Mode skills have `mode: true`.

## 14. Adapting to your environment — drop-in replacements

| Original (Cursor) | Any-agent replacement |
|---|---|
| `subagent_type: poteto-agent` / `Comment Sicko` | generic subagent; pass role + constraints in prompt |
| `~/.cursor/rules/pstack-models.mdc` roles + slugs like `grok-4.6-fast-xhigh`, `claude-fable-5-thinking-max` | pick `fast code model` for mechanical edits, `strong reasoning model` for design/judgment, `different family` for judge; or omit → inherit parent |
| `environment: cloud` vs `local` | `local` when need worktree/simulator/local transcripts; else isolated temp dir |
| MCP categories (Linear/Notion/Slack/Datadog/Sentry/Databricks) | map to whatever you have: git/gh + tracker + docs + chat + observability + error tracking + analytics; unavailable → skip with explicit justification, null result still reported |
| `cursor-team-kit` `control-ui`/`control-cli`, `deslop` | use your harness: browser/CDP, PTY, curl; deslop ≈ remove dead guards/compat/narrating comments/unrelated edits + lint |
| Graphite `gt` stacking, `orch/orch.ts`, `watch-pr` | any stacking: branches + rebase; track PR list + SHAs manually or via `gh`; frontier = ordered PR list + SHAs + generation |
| `/loop`, `/goal`, `AskQuestion` | your loop/heartbeat mechanism + explicit done predicate; questions only for genuine product calls |
| `agent-transcripts/` | your transcript/log location; if none, use task log + decisions.tsv |

You don't need to port scripts to use the skill. The workflows run fine with `git`, `gh`, shell, and any subagent/task tool. Port a script only when its lever saves repeated work.

---

**Attribution:** this file distills `cursor/plugins` `pstack` guide + `poteto-mode` + all playbooks + all principle/leaf skills, de-Cursored and compressed so any agent can read one file and run the workflow. Keep the principles; copy playbook steps verbatim into todos; prove it works on the real artifact.

