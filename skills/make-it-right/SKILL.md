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
spawn them by name; with plain subagents, paste the role into the prompt; with
none, perform each role yourself in order. The steps stay the same. A step that
spawns subagents is done only when their results are in: wait for each one in
the foreground, since a result that lands after your turn ends is lost. Where
the spawn tool runs in the background by default, ask for foreground on every
spawn.

When the harness lets you pick a model per subagent, pick one on every
spawn, at every scope size:

- **fast and cheap:** scouts, `naming`, and bulk mechanical edits such as
  wide renames or reference sweeps;
- **balanced:** `duplication` and `idioms-and-concurrency` proposers, and
  applying straightforward proposals;
- **strongest:** `structure-coupling`, synthesis, `behavior-verifier`, and any
  edit or verification touching concurrency or a public surface, whatever the
  dimension.

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
- **Size** counts the whole scope, not each package in it; a slice of a large
  scope (section 4) is sized on its own files. For *small* scopes (about ten
  files and two thousand lines at most; past either limit the scope is medium),
  scout yourself, spawn one `refactor-proposer` for all four dimensions, and one
  `behavior-verifier`. *Medium* scopes (more than that, up to about one package
  or five thousand lines) use the full fan-out below. *Large* scopes and `all`
  split into slices (section 4). A larger tier named by the caller wins over the
  count.

## 2. Gate and baseline

The gate is a **ratchet**: a commit may not add a failure. It need not fix
failures that were already there.

- Find the commands in the repo's agent docs (AGENTS.md, CLAUDE.md,
  CONTRIBUTING) and task-runner config. Use two tiers. The **targeted gate** is
  lint, typecheck, and the tests covering touched files and their callers, run
  after each change. Find those tests by searching the whole test tree for
  imports of each touched module; they often live outside its package. When that
  is most of the suite, take the direct importers, the module's own suites, and
  any static guards, and widen to the callers of every helper you add in another
  module. The **full gate** is the project's check command, run once before the
  first edit and once at the end. If the docs map change types to checks, follow
  that map.
- Before the first edit, record the baseline: run the full gate and list failing
  checks or tests by name, unless a baseline was handed to you for this commit
  or for an ancestor whose later commits touch nothing the gate runs. An
  inherited baseline covers only the checks it names: a failure in any other
  check is reported with its attribution, not ratcheted. When the full gate
  stops at its first failure, run its remaining parts separately so the baseline
  names every failing check. A red baseline is recorded, not a stop. From then
  on, green means no failure outside the baseline list. If the gate cannot
  produce a signal at all (missing tool, broken install, collection error), or
  nothing runnable exists, fix mode becomes review mode and the report says why.
- Gate runs can outlast a single tool call. Run a long gate in the background,
  in its own session (`setsid`) so a harness timeout cannot kill it, with its
  exit code and process ID written to files (stop it by that ID, since a pattern
  kill can match your own shell), then poll for the exit-code file, and when a
  wait times out, wait again. Before each launch, stop any earlier run still
  alive by its recorded ID and delete its exit-code file, so the poll never
  reads an earlier run's code. A run is finished only when its exit code exists
  (a notice that the background shell exited is not that); continue or end your
  turn only after that.
- Review mode needs no baseline. Run targeted tests only when a finding
  depends on runtime behavior.
- Pre-existing uncommitted changes (what `git status` shows) belong to the
  user. Leave them exactly as they are. Prefer a separate worktree or branch
  for the pass. Otherwise, skip proposals that touch dirty files and list them
  in the report, and stage your own edits by path.

## 3. The pass

Track this checklist in your response, once per slice:

```
- [ ] Scout    - [ ] Propose    - [ ] Synthesize
- [ ] Apply (fix)    - [ ] Verify (fix)    - [ ] Report
```

Tick a step when its size tier's count has run: on a medium or large slice,
two scouts, four proposers, and two verifiers. A step run short or skipped
stays unticked, and the report says why.

1. **Scout** (parallel, read-only; any read-only search agent will do). One
   subagent hunts existing helpers, utilities, and types that the scoped code
   duplicates. The other compares each scoped module with 2-3 neighbours of the
   same kind and lists where it deviates from their conventions. Done when every
   scoped module has its neighbours named, plus each duplicated helper or "none
   found".
2. **Propose** (parallel, read-only). Spawn one `refactor-proposer` per
   dimension (one for all four on a small scope): `naming`, `duplication`,
   `structure-coupling`, `idioms-and-concurrency`. Give each the file list,
   the goal of the change (from commit messages or the PR description), the
   scout findings, and the project notes.
3. **Synthesize.** Merge and dedupe across dimensions. Check each proposal's
   claims of fact (dead code, no other callers, identical bodies) by search
   before accepting it: a proposer's claim is a lead, not evidence. Reject
   anything that would change observable behavior, public API, route paths,
   serialized field names, test expectations (a test's patch targets, imports,
   and the names it calls follow the code they name; what an assertion expects
   stays), or what loads when (a module newly loaded at startup, a bundle chunk
   split or merged), or how long a resource lives (a client, pool, or lock now
   kept, shared, or closed elsewhere). Also reject new dependencies, dedupes of
   code that only looks alike, and proposals with no named payoff. Put a shared
   helper in the lower-level module, so no low-level module starts importing a
   higher layer, and in one whose own imports its callers already load. A hazard
   fix that changes what fails, throws, or logs (a swallowed error now caught, a
   race closed) is a behavior change: list it under human judgment rather than
   applying it. When proposals conflict, keep the smaller change unless the
   larger one subsumes it. Order the rest: renames, then dedupes, then moves and
   splits, so extracted helpers get good names. Done when every proposal is
   accepted or rejected with a reason. In fix mode, write the accepted list with
   its file:line anchors to a scratch file outside the repository and apply from
   that file, so the detail survives a long run. **Review mode ends here:**
   report the accepted list as findings, each with file:line, the concrete
   change, and its risk.
4. **Apply** (fix mode, serially). Edit yourself or through one subagent at a
   time; the commits stay yours. For each accepted proposal, finishing its
   commit before editing for the next:
   - make the edit and update every reference by search, including strings,
     docs, scripts, and dynamic dispatch the type checker cannot see; done
     when a search for the old name or path finds only intended hits;
   - run the targeted gate on the uncommitted edit (small proposals on
     disjoint files may share one run, each still in its own commit);
   - when it is green, commit that proposal alone, or with others of its kind in
     the same file (`refactor: rename X to Y`, `refactor: dedupe X into Y`,
     `refactor: extract X`, `refactor: move X to Y`);
   - when it is red, find each part that failed (lint, types, or a test) by
     reading the log from the top, since a tail shows only the last, before
     reading test counts. Finish the change it points at (a line the rename
     pushed past the length limit, a stale patch-target string), or restore this
     proposal's files to `HEAD` and retry a smaller version once, or skip it and
     record why.
5. **Verify** (fix mode). Run the full gate against the baseline. Then spawn
   `behavior-verifier` on `<start>..HEAD` (two independent verifiers for medium
   and large scopes). On a **changed** verdict, fix or revert that commit and
   verify again. On a **suspect** verdict, resolve it to preserved with
   evidence, fix it, or revert it; revert when no measurement can settle it,
   such as a race under load. When verifiers disagree, measure the disputed fact
   (run it, build it, diff the output) rather than pick a side. A **preserved**
   verdict that names a defect in the refactor itself (code it left orphaned, a
   comment it made false) still gets a fix commit or a recorded reason. After
   any fix or revert, rerun the full gate. Done when every commit in
   `<start>..HEAD` is preserved with evidence, including commits made after the
   verifiers ran.
6. **Report.** Include the mode, scope, and baseline; commits (hash and
   subject) or findings; proposals rejected or skipped and why; verifier
   verdicts; which roles you performed yourself instead of spawning; and
   what needs human judgment, such as public API changes,
   cross-package moves, or unproven behavior differences. Leave pushing and
   merging to the user.

## 4. Slices (large scopes and `all`)

Split the scope into slices of disjoint files along module or package
boundaries, each about the size of one reviewable PR (a few hundred changed
lines). A file too big for one slice is a slice of its own; split its work by
section. Rank them by **hotspot**: churn (commits in the last few months) times
size. Each slice is its own pass through section 3, with its own report and
branch, and at most a medium scope; its own size sets its tier. Slices worked
serially in one worktree may share one scout and propose round, ordered by
slice. Slices run in parallel only in separate worktrees; in a shared worktree
they run serially. Duplication that spans slices goes into a final cross-slice
pass once the slices land. Slices cut from one commit share one baseline: record
it once and hand it to each slice.
