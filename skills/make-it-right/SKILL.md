---
name: make-it-right
description: Behavior-preserving cleanup of working code - naming, duplication, structure and coupling, idioms and concurrency - over a branch, a PR, paths, or the whole codebase. Review mode returns ranked findings without editing, so it fits code review; fix mode applies them as gated refactor commits, then verifiers try to prove behavior changed. Use when asked to make it right, clean up, refactor, or tidy code, to review a diff for code quality, or once a feature works and before it merges.
---

Same behavior, better code. Subagents propose (read-only), you synthesize and
apply serially behind a **ratchet** gate, and adversarial verifiers try to
prove behavior changed before you report success.

Read the project notes first: `.agents/skills/make-it-right/PROJECT.md` in
the repository, and `PROJECT.md` next to this file if present. Their gates,
exclusions, and conventions override the defaults here.

Roles live in `roles/` next to this file. With named subagents, register and
spawn them by name; with plain subagents, paste the role into the prompt;
with none, perform each role yourself in order. The steps stay the same.

When the harness lets you pick a model per subagent, pick one on every
spawn, at every scope size:

- **fast and cheap:** scouts, `naming`, and bulk mechanical edits such as
  wide renames or reference sweeps;
- **balanced:** `duplication` and `idioms-and-concurrency` proposers, and
  applying straightforward proposals;
- **strongest:** `structure-coupling`, synthesis, `behavior-verifier`, and
  anything touching concurrency or a public surface.

## 1. Frame the run

Decide these from the request and the repo, state them at the top of your
report, and ask only when the request is genuinely ambiguous:

- **Mode.** `review` returns findings and touches nothing: no edits, staging,
  or commits. Use it for code review, someone else's branch or PR, or "what
  would you clean up?". `fix` edits and commits. Use it when asked to clean
  up, refactor, or make right the current working branch.
- **Scope.** Arguments win: a PR or branch means its diff against its base,
  paths mean every file under them, changed on the branch or not, and `all`
  means the whole repository. Otherwise use
  the current branch's diff against its merge-base with the main branch. If
  that diff is empty, say so and offer a path or `all`. Drop vendored,
  generated, and lockfile paths.
- **Size.** For *small* scopes (about ten files or fewer), scout yourself,
  spawn one `refactor-proposer` for all four dimensions, and one
  `behavior-verifier`. *Medium*
  scopes use the full fan-out below. *Large* scopes and `all` split into
  slices (section 4).

## 2. Gate and baseline

The gate is a **ratchet**: a commit may not add a failure. It need not fix
failures that were already there.

- Find the commands in the repo's agent docs (AGENTS.md, CLAUDE.md,
  CONTRIBUTING) and task-runner config. Use two tiers. The **targeted gate**
  is lint, typecheck, and the tests covering touched files and their callers,
  run after each change. Find those tests by searching the whole test tree
  for imports of each touched module; they often live outside its package. The **full gate** is the project's check command, run
  once before the first edit and once at the end. If the docs map change
  types to checks, follow that map.
- Before the first edit, record the baseline: run the full gate and list
  failing checks or tests by name. When the full gate stops at its first
  failure, run its remaining parts separately so the baseline names every
  failing check. A red baseline is recorded, not a stop.
  From then on, green means no failure outside the baseline list. If the
  gate cannot produce a signal at all (missing tool, broken install,
  collection error), or nothing runnable exists, fix mode becomes review mode
  and the report says why.
- Gate runs can outlast a single tool call. Wait for each run to finish
  before you continue or end your turn.
- Review mode needs no baseline. Run targeted tests only when a finding
  depends on runtime behavior.
- Pre-existing uncommitted changes belong to the user. Leave them exactly as
  they are. Prefer a separate worktree or branch for the pass. Otherwise,
  skip proposals that touch dirty files and list them in the report, and
  stage your own edits by path.

## 3. The pass

Track this checklist in your response:

```
- [ ] Scout    - [ ] Propose    - [ ] Synthesize
- [ ] Apply (fix)    - [ ] Verify (fix)    - [ ] Report
```

1. **Scout** (parallel, read-only). One subagent hunts existing helpers,
   utilities, and types that the scoped code duplicates. The other compares
   each scoped module with 2-3 neighbours of the same kind and lists where it
   deviates from their conventions. Done when every scoped module has its
   neighbours named, plus each duplicated helper or "none found".
2. **Propose** (parallel, read-only). Spawn one `refactor-proposer` per
   dimension: `naming`, `duplication`, `structure-coupling`,
   `idioms-and-concurrency`. Give each the file list, the goal of the change
   (from commit messages or the PR description), the scout findings, and the
   project notes.
3. **Synthesize.** Merge and dedupe across dimensions. Reject anything that
   would change observable behavior, public API, route paths, serialized field
   names, or test assertions. Also reject new dependencies, dedupes of code
   that only looks alike, and proposals with no named payoff. Put a shared
   helper in the lower-level module, so no low-level module starts importing
   a higher layer. When proposals conflict, keep the smaller change.
   Order the rest: renames, then dedupes, then moves and splits, so extracted
   helpers get good names. Done when every proposal is accepted or rejected
   with a reason. **Review mode ends here:** report the accepted list as
   findings, each with file:line, the concrete change, and its risk.
4. **Apply** (fix mode, serially). For each accepted proposal:
   - make the edit and update every reference by search, including strings,
     docs, scripts, and dynamic dispatch the type checker cannot see;
   - run the targeted gate;
   - when it is green, commit one refactoring kind (`refactor: rename X to Y`,
     `refactor: extract X`, `refactor: move X to Y`);
   - when it is red, restore only this proposal's files, retry a smaller
     version once, or skip it and record why.
5. **Verify** (fix mode). Run the full gate against the baseline. Then spawn
   `behavior-verifier` on `<start>..HEAD` (two independent verifiers for
   medium and large scopes). On a **changed** verdict, fix or revert that
   commit and verify again. On a **suspect** verdict, resolve it to preserved
   with evidence, or revert. Done when every commit is preserved with
   evidence.
6. **Report.** Include the mode, scope, and baseline; commits (hash and
   subject) or findings; proposals rejected or skipped and why; verifier
   verdicts; and what needs human judgment, such as public API changes,
   cross-package moves, or unproven behavior differences. Leave pushing and
   merging to the user.

## 4. Slices (large scopes and `all`)

Split the scope into slices of disjoint files along module or package
boundaries, each about the size of one reviewable PR. Rank them by
**hotspot**: churn (commits in the last few months) times size. Each slice
is its own pass through section 3, with its own report and branch. Slices
run in parallel only in separate worktrees; in a shared worktree they run
serially. Duplication that spans slices goes into a final cross-slice pass
once the slices land. Slices cut from one commit share one baseline:
record it once and hand it to each slice.
