---
name: maintain
description: Phase 4 maintenance pass - fans out maintenance-auditor agents across dependencies, security alerts, docs drift, and coverage gaps, then applies the merged work-list serially behind the project's check gate. Use on a regular cadence (weekly), when asked to do repo maintenance, update dependencies, or sync docs. The final report doubles as a PR body for scheduled runs.
---

Orchestrate a maintenance pass. Auditor subagents produce work-lists in
parallel (read-only); you apply serially with per-category commits behind the
project's check gate; risky items are deferred to the human report, never
applied silently.

Harness note: role playbooks live in `roles/` next to this file. If your
harness supports named subagents, register the roles and spawn them by name in
parallel. If it supports plain subagents, paste each role playbook into the
subagent's instructions. If it has no subagents at all, perform each role
yourself, sequentially. The stages and gates below stay identical either way.

0. **Find the gates.** Locate the project's check gate (canonical lint +
   typecheck + test command) and its stricter pre-merge gate if one exists —
   see the repo's agent/contributor docs and task-runner config. Dependency
   changes always use the strictest gate available.

1. **Baseline.** Run the check gate. If red, stop — maintenance never starts
   from a broken state; report the failure instead. Note the current branch;
   if on the main branch, create `maintenance/<date>` first.

2. **Audit fan-out (parallel, read-only).** Run 4 subagents with the
   maintenance-auditor role (`roles/maintenance-auditor.md`), one per
   category:
   - `deps`: outdated dependencies and stale tool/scanner pins — minus
     whatever the repo's existing update automation (Renovate, Dependabot, a
     custom workflow) already owns;
   - `security-alerts`: open alerts on the repo host (skips cleanly if the
     host CLI is unauthenticated);
   - `docs-drift`: agent/contributor docs and docs/ vs the actual commands,
     module layout, and endpoints;
   - `coverage-gaps`: the test-coverage report, business-critical modules
     only.
   Each returns a prioritized work-list with effort and risk per item.

3. **Plan.** Merge the work-lists. Apply-now = low/medium risk, small/medium
   effort. Defer-to-report = major version bumps, pinned tools with an
   explanatory comment, anything touching public API or infrastructure,
   large-effort items. When in doubt, defer.

4. **Apply serially, per category:**
   - **deps**: leave routine bumps to the repo's automation if it has any;
     apply only targeted items it can't handle (a security-motivated bump of
     one package, stale scanner pins the auditor flagged). Review the
     lockfile diff for surprises, gate with the strictest check available,
     commit `chore(deps): ...`.
   - **security-alerts**: fix per the harden skill's rules (refute first,
     High first), commit `fix(security): ...`.
   - **docs-drift**: correct the stale side (usually docs; if the code is the
     odd one out, report it instead), commit `docs: ...`.
   - **coverage-gaps**: add tests for the chosen uncovered branches, extend
     existing test files/fixtures, run the new tests directly first, then the
     check gate, commit `test: ...`.
   A category with nothing applicable is fine — say so in the report.

5. **Verify.** If any lockfile change shipped, run one subagent with the
   behavior-verifier role (`roles/behavior-verifier.md`) over the full diff
   to hunt for behavior drift from upgraded deps (changed defaults,
   deprecations newly triggered at load time). Any **changed** verdict:
   revert that upgrade and defer it. For a **suspect** verdict, investigate
   until it resolves to preserved-with-evidence; otherwise revert and defer the
   upgrade. Never ship an unresolved suspect dependency change.

6. **Report** (this is the PR body on scheduled runs): per category — what was
   applied (commits), what was deferred and why (with the exact command a
   human would run), and auditor findings that need discussion. End with the
   gate status. Do not push or merge unless asked; on a `maintenance/*`
   branch, offer the PR command instead.
