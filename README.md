# pstack-universal — any-agent skill

> Universal, agent-agnostic reverse-engineering of `cursor/plugins` → `pstack` (`poteto-mode`). One file any agent can read and follow — no Cursor, no MCPs, no specific models required.

**Fork source:** [`cursor/plugins` @ `main` → `pstack`](https://github.com/cursor/plugins/tree/main/pstack) → forked to [`immortaldarkveil/plugins`](https://github.com/immortaldarkveil/plugins)

**What it is:** 157 files / 7800+ lines / 22 playbooks / 21 principles / ~40 leaf skills distilled into **one 557-line `SKILL.md` (53KB)** any LLM agent can load.

Any agent that can read markdown, run shell commands (`git`, `gh`), and delegate subtasks can follow it: OpenCode, Cursor, Cline, Aider, Codex, Claude Code, etc.

---

## Install

### Opencode

```bash
# clone into your opencode skills dir or copy the file
git clone https://github.com/immortaldarkveil/pstack-universal.git
cp -r pstack-universal/.opencode/skills/pstack-universal ~/.config/opencode/skills/
# or project-local
cp -r pstack-universal/.opencode/skills/pstack-universal .opencode/skills/
```

### Any agent / any repo

```bash
git clone https://github.com/immortaldarkveil/pstack-universal.git
# or just copy the one file
curl -o SKILL.md https://raw.githubusercontent.com/immortaldarkveil/pstack-universal/main/SKILL.md
```

Tell your agent to read it at start of any multi-step task:

```
Read SKILL.md (or .opencode/skills/pstack-universal/SKILL.md) in full before you start. Follow its dispatcher.
```

### Generic discovery

The skill is duplicated for discoverability:

- `./SKILL.md` — root (any agent scanning repo)
- `./.opencode/skills/pstack-universal/SKILL.md` — opencode-native
- `./skills/pstack-universal/SKILL.md` — generic `skills/` convention

All three are identical.

---

## How to use (human side)

Stop micromanaging the agent. Describe **what you want** + **how you'll know it's done**. The skill's dispatcher picks the playbook and sequences the work.

**Examples:**

```text
# Bug
the export writes duplicate rows when a retry lands mid-run. repro first, then fix and verify.

# Feature
add a --json flag. text output stays byte-identical. verify both.

# Refactor
move parsing into one module, zero behavior change. record output before and prove unchanged after.

# Perf
startup takes 1.8s on this fixture. trace it, fix measured cause, show before/after.

# Research (read-only)
how does the sync service decide ownership? don't change code yet.

# Design
design the import pipeline before writing code. i care most about caller usage.

# Overnight / autonomous
im stepping away. keep going until migration check reports zero old callers. log decisions. don't ask before committing.
```

You don't name playbooks or list skills. A phrase like "repro first" or a checkable predicate is enough. To switch tasks mid-chat: `new task. ...`.

See `SKILL.md` Section 2 and Section 12 for the full prompt recipes.

---

## What the skill enforces

**Non-negotiables:** first todo is always `read Principles in full` → each principle citation must map to a decision. Code crossing a function boundary → `architect` + parallel design via `arena`. Parallel fan-out → `swarm`/`arena`. Contested design → `interrogate` before ship. Any prose → `unslop`. Any PR → `technical-writing` + `deslop` + `no-comments`. Long runs → `show-me-your-work` TSV trail.

**21 principles distilled:** Laziness Protocol, Foundational Thinking, Redesign from First Principles, Subtract Before You Add, Minimize Reader Load, Outcome-Oriented, Experience First, Exhaust Design Space, Build the Lever, Model the Domain, Boundary Discipline, Type System Discipline, Make Operations Idempotent, Migrate Callers Then Delete, Separate Before Serializing, Prove It Works, Fix Root Causes, Sequence Verifiable Units, Guard Context Window, Never Block on the Human, Encode Lessons in Structure — each with `when → rule → test`.

**Playbooks distilled (copy verbatim into todos):** Investigation, Bug fix, Feature, Refactoring, Prototype, Perf issue, Hillclimb, Runtime/Trace Forensics, Visual parity, Eval, Multi-phase plan, Autonomous run, Orchestrate, Babysit, Shipping, Opening a PR, Session pickup/Pause, Worktree cleanup.

**Leaf skills distilled:** `how` (explorer→explainer), `why` (7-evidence parallel), `teach`/`recall`/`architect`/`arena` (base+graft+judge)/`swarm`/`interrogate`/`tdd`/`blast-radius`/`unslop`/`no-comments`/`figure-it-out`/`show-me-your-work`/`create-verification-skill`.

**Verification contract:** `prove-it-works` on the real artifact (not "it compiles") + `sequence-verifiable-units` (red failing artifact before green fix, verify each unit before next).

Details in `SKILL.md` Sections 3–11.

---

## Verifying the distillation

Source mapped at clone time:

```
# original playbooks
pstack/skills/poteto-mode/playbooks/*.md  # 22 playbooks
pstack/skills/principle-*/SKILL.md        # 21 principles
pstack/skills/*/SKILL.md                  # ~40 leaf skills
pstack/docs/guide/*.md                    # 10 guide pages
```

This repo's `SKILL.md` was checked to cover all 21 principles and 13 playbook families + 13 leaf skills (`default.bash` grep checks in build log).

Original license: MIT © 2026 Lauren Tan (see `LICENSE`).

---

## Adapting to your env

No Cursor needed. Drop-in replacements in `SKILL.md` Section 14:

| Original | Any-agent |
|---|---|
| `poteto-agent` / model slugs (`grok-4.6-fast-xhigh`, `claude-fable-5-thinking-max`) | pick `fast code` vs `strong reasoning` vs `different family` for judge, or inherit parent |
| MCPs (Linear/Notion/Slack/Datadog/Sentry/Databricks) | use whatever you have: `git`/`gh` + tracker + docs + chat + observability + error tracking + analytics; report null results |
| `control-ui`/`control-cli`/`gt`/`/loop` | browser CDP / PTY / `curl` + branches + `gh` + your heartbeat predicate |
| `agent-transcripts/` | your transcript/log dir or `decisions.tsv` |

---

## License

MIT — same as `cursor/plugins` `pstack`. See `LICENSE`.

Attribution: distilled from [`cursor/plugins`](https://github.com/cursor/plugins) `pstack` guide + `poteto-mode` + all playbooks + all principle/leaf skills, de-Cursored and compressed so any agent can read one file and run the workflow. Keep the principles; copy playbook steps verbatim into todos; prove it works on the real artifact.
