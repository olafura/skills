---
name: harden
description: Phase 3 security and performance pass - runs the project's security scanners, fans out security-adversary agents across the system's attack surfaces, refutes findings adversarially before fixing High-first, and profiles before optimizing. Use when asked to harden, security-review, audit, or optimize the project, or after make-it-right on a security-relevant change.
---

Orchestrate a security (and optionally performance) pass. Subagents find and
refute in parallel (read-only); you fix serially behind the project's check
gate; nothing gets reported as fixed until the scanners agree.

Harness note: role playbooks live in `roles/` next to this file. If your
harness supports named subagents, register the roles and spawn them by name in
parallel. If it supports plain subagents, paste each role playbook into the
subagent's instructions. If it has no subagents at all, perform each role
yourself, sequentially. The stages and gates below stay identical either way.

0. **Find the gates.** Locate the project's check gate (canonical lint +
   typecheck + test command — see the repo's agent/contributor docs and
   task-runner config) and its security-scan command if one exists. If there
   is no runnable check gate, stop and report — never harden without a gate. If
   there is no security-scan command, pick the standard scanners for the
   ecosystem — a static analyzer plus a dependency-vulnerability auditor (e.g.
   bandit + pip-audit, npm/pnpm audit + semgrep, cargo-audit, govulncheck,
   brakeman — whatever fits the stack) — run them via ephemeral/pinned
   invocations, and note in the report that the project should adopt a
   permanent scan task.

1. **Baseline and mechanical scan.** Run the check gate before making any
   edits. If it is red, stop and report the failure — never harden on a broken
   baseline. Then run the scanners and collect every finding (a nonzero exit
   with findings is expected, not an error). Scope note: if this skill's
   arguments name files/areas, still run the full scan but prioritize findings
   in scope.

2. **Learn the trust model, then sweep (parallel, read-only).** First read
   what the repo says about its own security posture (security docs,
   architecture docs, deployment config): what is internet-facing vs internal,
   what is authenticated where, what is fail-closed by design. Then pick 3-5
   attack surfaces and run one subagent with the security-adversary role
   (`roles/security-adversary.md`) per surface, each blind to the others.
   Default surfaces when the repo suggests nothing better:
   - authentication and authorization boundaries (every lane: user, service,
     admin, webhook);
   - input handling (parsing, deserialization, injection, anything reaching
     the system from users or third parties);
   - secrets, configuration, and logging (leakage, staleness, cache keys);
   - third-party and network boundaries (outbound calls, isolation between
     tenants/integrations, SSRF surface);
   - data exposure (what responses, errors, and artifacts reveal).
   Give each subagent the trust model so it flags real gaps, not boilerplate.

3. **Refute pass (parallel, read-only).** Merge scanner + sweep findings,
   dedupe by file/issue. For each High/Critical (and any Medium that looks
   plausible), run one verifier subagent prompted to REFUTE it: is it actually
   reachable given the deployment boundary and the design's existing
   mitigations? Default to refuted when the exploit path can't be
   demonstrated. Only confirmed findings proceed; record refuted ones.

4. **Fix serially.** Confirmed findings, High/Critical first, then Medium:
   - fix the root cause, not the symptom;
   - run the check gate after each fix; commit each separately
     (`fix(security): ...`);
   - genuine scanner false positives: suppress with the tool's inline
     mechanism plus a justification comment; unfixable-now dependency vulns:
     record for the report, don't silently ignore.
   Re-run the scanners until they exit clean (or only carry findings
   explicitly deferred in the report).

5. **Platform alerts.** If the repo host's CLI is authenticated (gh, glab,
   ...), pull open security alerts (code scanning, dependency alerts, secret
   scanning) and triage them the same way: refute first, fix High-first.
   Dependency-alert fixes are targeted version bumps, gated by the project's
   full pre-merge check before committing (`fix(deps): ...`).

6. **Perf lane — only when warranted.** Only if this skill's arguments name a
   hotspot or the sweep surfaced a concrete performance problem: capture a
   baseline with the project's benchmark/profiling command (or the ecosystem's
   standard profiler), then run one subagent with the performance-analyst role
   (`roles/performance-analyst.md`) on the capture — it returns ranked,
   evidence-backed proposals, no edits. Apply serially behind the check gate
   (one `perf:` commit each), re-measure, and keep only changes whose number
   moved. Proposals the analyst flagged as behavior-risky get a
   behavior-verifier pass (`roles/behavior-verifier.md`). A **changed** verdict
   means revert that performance commit. For a **suspect** verdict, investigate
   until it resolves to preserved-with-evidence; otherwise revert the commit.
   Never retain an unresolved suspect optimization. No speculative
   optimization.

7. **Verify the final tree.** After all platform-alert and performance work,
   run the check gate and the full scanner set again against the final source
   and dependency tree. Fix or revert any new non-deferred failure, then repeat
   until the gate is green and scanners exit clean (or carry only findings
   explicitly deferred in the report). Do not rely on results from step 4.

8. **Report.** A findings table: finding → source (scanner / sweep / platform
   alert) → verdict (confirmed / refuted with reason) → resolution (fixed
   commit / suppressed with justification / deferred with owner-worthy
   detail). Include the final check-gate and scanner commands and results, plus
   benchmark before/after numbers if the perf lane ran. Do not push or merge.
