# Skills

Agent-agnostic skills for AI coding assistants, in the
[skills.sh](https://www.skills.sh) layout. Each skill is a plain-markdown
playbook (`skills/<name>/SKILL.md`) any harness can run — Claude Code, Cursor,
Codex, OpenCode, or a human following it by hand. Nothing here assumes a
specific language, framework, or task runner: every skill starts by
discovering *your* project's verification commands and uses those as its
gates.

## Installing

Install skills individually via the [skills.sh](https://www.skills.sh) CLI —
pick the ones you need:

```bash
npx skills add https://github.com/olafura/skills --skill make-it-right
npx skills add https://github.com/olafura/skills --skill harden
npx skills add https://github.com/olafura/skills --skill maintain
```

The skills reference each other (e.g. `maintain` triages security alerts per
`harden`'s rules, and `harden` follows `make-it-right`), so installing the
whole group together works best — but each one runs standalone.

Or wire them in manually — copy or symlink a skill directory into your
agent's skill location (e.g. `.claude/skills/` for Claude Code, or your
harness's equivalent). Each skill ships its subagent role playbooks in a
`roles/` folder next to its SKILL.md, so a skill directory is self-contained.

## Phased Delivery

> **Make it Work → Make it Right → Make it Fast & Secure → Keep it Working**

https://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast

**The idea:** AI assistants make Phase 1 — getting something working — nearly
free, which makes it dangerously easy to stop there. Code that only ever
reaches "it works" piles up into unclear names, duplicated logic, an
unaudited security posture, and drifting docs. The classic answer (Kent Beck,
rephrased by Joe Armstrong as *"make it work, then make it beautiful, then if
you really, really have to, make it fast"*) is to do **one thing at a time**.
These skills package phases 2–4 as explicit, repeatable agent passes so they
actually happen instead of staying good intentions.

All three share one orchestration invariant:

1. **Subagents propose** — specialists fan out in parallel, read-only, each
   owning one dimension or attack surface, and return ranked proposals with
   file:line anchors. They never edit.
2. **The orchestrator applies** — serially, one concern per commit, running
   the project's check gate (lint + typecheck + tests) after every commit.
   Red gate → revert, retry smaller, or skip.
3. **Adversaries verify** — before success is reported, independent verifier
   agents attack the result (trying to *prove* behavior changed, or to
   *refute* a security finding). Nothing ships on the proposing agent's word.

Phase 1 needs no skill — that's the part agents already do well. The gate for
Phase 1 is simply: the feature demonstrably works and the project's check
command is green.

| Phase | Skill | One-liner |
|---|---|---|
| 2 — Make it Right | [`make-it-right`](skills/make-it-right/SKILL.md) | Behavior-preserving cleanup of a working branch |
| 3 — Make it Fast & Secure | [`harden`](skills/harden/SKILL.md) | Adversarial security pass; performance only where measured |
| 4 — Keep it Working | [`maintain`](skills/maintain/SKILL.md) | Recurring deps / alerts / docs / coverage sweep |

### `make-it-right` — same behavior, better code

Run on a feature branch once it works. Two scouts map what the change touched
and what existing code it should have reused; four **refactor-proposer**
agents each own one dimension — naming, duplication (rule of three, and Sandi
Metz's *"duplication is far cheaper than the wrong abstraction"*),
structure/coupling, and language idioms & concurrency. The orchestrator
synthesizes (dropping anything behavior-risky), applies renames → dedupes →
moves as separate gated commits, and two **behavior-verifier** agents then
try to prove behavior changed — any confirmed change is fixed or reverted
before the pass reports success. Hard rules: never change observable
behavior, public API, or test assertions; no new dependencies; revert rather
than force a failing refactor.

### `harden` — audited security, measured performance

Three layers of skepticism. First the mechanical scanners (the project's
security-scan task, or the ecosystem's standard static analyzer + dependency
auditor). Then a sweep: the skill reads the repo's own trust model and fans
out **security-adversary** agents, one per attack surface, each thinking like
an attacker with source access — concrete exploit scenarios or it doesn't
count. Then a *refute pass* kills plausible-but-wrong findings before
anything is "fixed"; only confirmed findings get fixed, High-first, one
gated commit each, until the scanners come back clean. Performance is a
separate lane that only opens when a hotspot is named or found: benchmark,
hand the profile to a **performance-analyst** that refuses to propose
anything without measurement evidence, apply, re-benchmark, keep only what
moved the number.

### `maintain` — the codebase stays healthy after shipping

Run weekly (a scheduled CI job that opens a PR works well — the skill's final
report is written to double as the PR body). Four **maintenance-auditor**
agents audit in parallel: outdated dependencies (deferring to whatever update
automation the repo already has — Renovate, Dependabot, custom), open
security alerts on the repo host, docs drift (docs vs the actual commands and
layout), and coverage gaps in business-critical code. Low-risk items are
applied behind the gates with per-category commits; major bumps, commented
pins, and anything touching public API or infra are deferred to the human
report. If a lockfile changed, a **behavior-verifier** hunts the diff for
behavior drift from upgraded deps.

### The roles

Each skill vendors the role playbooks it spawns under `skills/<name>/roles/`,
so an installed skill is self-contained (`behavior-verifier` is intentionally
duplicated into every skill that uses it — if you edit it, sync the copies):

| Role | Used by | Character |
|---|---|---|
| `refactor-proposer` | make-it-right | One dimension, ranked proposals, never edits |
| `behavior-verifier` | all three | Tries to *prove* a "safe" diff changed behavior; defaults to suspicion |
| `security-adversary` | harden | One attack surface, trust-model-aware, exploit scenario or it doesn't count |
| `performance-analyst` | harden | Profile evidence before opinion; 80/20 ranking; caps at 5 proposals |
| `maintenance-auditor` | maintain | One category, prioritized work-list with effort + risk per item |

If your harness supports named subagents, register the roles and spawn them in
parallel (restrict them to read-only tools if your harness supports tool
restrictions — their playbooks demand it either way). If it supports plain
subagents, paste the playbook into each subagent's instructions. If it has no
subagents at all, every SKILL.md degrades to a sequential checklist — same
stages, same gates. A human can follow one manually and end up at the same
place.

## What these skills expect from your project

Only one thing: a **check gate** — a canonical command covering lint,
typecheck, and tests (`make check`, `mise run check`, `npm run check`,
`just check`, …). Each skill's step 0 discovers it from your repo's docs and
task-runner config, composes one from the individual commands if no single
verb exists, and refuses to run ungated if there's nothing at all. Documenting
the gate in your repo's agent instructions (AGENTS.md / CLAUDE.md) makes
every pass faster and more reliable.

## Layout

```
skills.sh.json                      groupings manifest for skills.sh
skills/<name>/SKILL.md              the skill playbook (frontmatter: name + description)
skills/<name>/roles/<role>.md       subagent playbooks the skill spawns
```

More groups will appear here over time; `skills.sh.json` is where they get
organized.

## License

[MIT](LICENSE)
