---
name: maintain
description: Maintenance sweep - maintenance-auditor agents audit dependencies, security alerts, docs drift, and coverage gaps, then low-risk items are applied as gated per-category commits and risky ones deferred to a human report. Report mode audits without editing. Use on a regular cadence, for scheduled or headless runs, or when asked to do repo maintenance, update dependencies, triage alerts, sync docs, or fill coverage gaps. The report doubles as a PR body.
---

Audit in parallel, apply serially, defer anything risky. Auditor subagents
produce work-lists (read-only); you apply per-category commits behind a
**ratchet** gate. Deferred items go to the human report with the command a
human would run.

Read the project notes first: `.agents/skills/maintain/PROJECT.md` in the
repository, and `PROJECT.md` next to this file if present. Their automation,
gates, and pins override the defaults here.

Roles live in `roles/` next to this file. With named subagents, register and
spawn them by name; with plain subagents, paste the role into the prompt;
with none, perform each role yourself in order. The steps stay the same.

When the harness lets you pick a model per subagent, pick one on every
spawn: `deps` and `docs-drift` auditors on the fast and cheap tier;
`security-alerts` and `coverage-gaps` on the balanced tier;
`behavior-verifier` on the strongest.

## 1. Frame the run

State these at the top of the report:

- **Mode.** `report` (also called dry run) audits and writes the report,
  touching nothing. `fix` applies and commits. Headless and scheduled runs
  take the mode from their caller.
- **Categories.** All four by default (`deps`, `security-alerts`,
  `docs-drift`, `coverage-gaps`), or the ones named in the arguments.
- **Branch.** On the main branch, fix mode first creates
  `maintenance/<date>`.
- **Gates.** The check gate and the strictest pre-merge gate come from the
  repo's agent docs and task-runner config. Dependency changes always use the
  strictest gate.

## 2. Baseline

The gate is a **ratchet**: an applied item may not add a failure. Before the
first edit, run the check gate and record failing checks or tests by name.
When the gate stops at its first failure, run its remaining parts separately
so the baseline names every failing check. A red baseline is recorded, not
a stop: fixing it may itself be a work item. Gate runs can outlast a single
tool call; wait for each run to finish before you continue or end your
turn.
If the gate cannot produce a signal at all, fix mode becomes report mode and
the report says why. Pre-existing uncommitted changes belong to the user;
leave them exactly as they are and skip items that touch dirty files.

## 3. The pass

```
- [ ] Audit   - [ ] Plan   - [ ] Apply (fix)   - [ ] Verify (fix)   - [ ] Report
```

1. **Audit** (parallel, read-only). Spawn one `maintenance-auditor` per
   category:
   - `deps`: outdated dependencies and stale tool or scanner pins, minus what
     the repo's update automation (Renovate, Dependabot, a custom workflow)
     already owns;
   - `security-alerts`: open alerts on the repo host, skipped with a note when
     the host CLI is unauthenticated;
   - `docs-drift`: agent and contributor docs and `docs/` against the actual
     commands, module layout, and endpoints;
   - `coverage-gaps`: the coverage report, business-critical code only.
2. **Plan.** Merge the work-lists. Apply now: low or medium risk with small
   or medium effort. Defer: major version bumps, pins that carry an
   explanatory comment, anything touching public API or infrastructure, and
   large items. When in doubt, defer. Done when every item is apply-now or
   deferred with a reason. **Report mode skips to step 5.**
3. **Apply** (fix mode, serially, per category):
   - **deps**: leave routine bumps to the repo's automation. Apply only the
     targeted items it can't handle, such as a security-motivated bump of one
     package or a stale scanner pin. Read the lockfile diff for surprises,
     run the strictest gate, and commit `chore(deps): ...`.
   - **security-alerts**: refute first and fix High first, per the harden
     skill; commit `fix(security): ...`.
   - **docs-drift**: correct the stale side. That is usually the docs; when the
     code is the odd one out, report it instead. Commit `docs: ...`.
   - **coverage-gaps**: add tests for the chosen branches in the existing test
     files and fixtures. Run the new tests directly, then the gate, and commit
     `test: ...`.
   A category with nothing applicable is a fine result; say so.
4. **Verify** (fix mode). If any lockfile change shipped, spawn
   `behavior-verifier` on the full diff to hunt for drift from upgraded
   dependencies: changed defaults, and deprecations newly triggered at load
   time. Revert and defer the upgrade on **changed**, and on any **suspect**
   you cannot resolve to preserved with evidence. Then run the full gate
   against the baseline.
5. **Report** (the PR body for scheduled runs). Per category: what was
   applied (commits), what was deferred and why with the exact command a
   human would run, and auditor findings that need discussion. End with the
   gate command and result. Leave pushing and merging to the user unless the
   caller says otherwise; on a `maintenance/*` branch, offer the PR command.
