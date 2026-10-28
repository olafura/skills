---
name: harden
description: Security and performance pass - runs the project's security scanners, sends security-adversary agents at the system's attack surfaces, refutes findings before acting, fixes confirmed ones High-first, and optimizes only what a profile shows. Review mode reports refuted-or-confirmed findings without editing, so it fits security review of a PR; fix mode applies gated fixes. Use when asked to harden, security-review, threat-model, audit, or optimize code, or after make-it-right on a security-relevant change.
---

Find, refute, then fix. Subagents find and refute in parallel (read-only);
you fix serially behind a **ratchet** gate. A finding counts as fixed once
the scanners and gate agree.

Read the project notes first: `.agents/skills/harden/PROJECT.md` in the
repository, and `PROJECT.md` next to this file if present. Their scanners,
trust model, surfaces, and gates override the defaults here.

Roles live in `roles/` next to this file. With named subagents, register and
spawn them by name; with plain subagents, paste the role into the prompt; with
none, perform each role yourself in order. The steps stay the same. A step that
spawns subagents is done only when their results are in: wait for each one in
the foreground, since a result that lands after your turn ends is lost. Where
the spawn tool runs in the background by default, ask for foreground on every
spawn.

When the harness lets you pick a model per subagent, pick one on every
spawn: scanner triage on the fast and cheap tier; refuters on the balanced
tier; adversaries, the performance analyst, and verifiers on the strongest.

## 1. Frame the run

State these at the top of your report, and ask only when the request is
genuinely ambiguous:

- **Mode.** `review` ends after the refute pass with a findings table and
  touches nothing. Use it for security review of a PR or branch, or "is this
  safe?". `fix` edits and commits. Use it when asked to harden the current
  working branch or the system.
- **Scope.** A PR, branch, or paths narrow the sweep to the surfaces they
  touch. With no scope, sweep the whole system.
- **Gates.** The check gate (lint, typecheck, tests) and the security scan
  come from the repo's agent docs and task-runner config. With no scan task,
  run the ecosystem's standard static analyzer plus a dependency auditor
  (for example bandit + pip-audit, semgrep + npm audit, cargo-audit,
  govulncheck), pinned and ephemeral, and recommend a permanent task in the
  report.

## 2. Baseline

The gate is a **ratchet**: a fix may not add a failure or a scanner finding.
It need not clear unrelated ones.

- Before the first edit, run the check gate and the scanners and record
  failing tests and every scanner finding. Scanners exiting nonzero on
  findings is the expected result. When the gate stops at its first
  failure, run its remaining parts separately so the baseline names every
  failing check. A red baseline is recorded, not a stop; if the gate cannot
  produce a signal at all, fix mode becomes review mode and the report says
  why.
- Gate and scanner runs can outlast a single tool call. Run a long one in the
  background, in its own session (`setsid`) so a harness timeout cannot kill it,
  with its exit code and process ID written to files by the run itself (a
  launcher's ID can belong to a parent that has already exited; stop the run by
  its own ID, since a pattern kill can match your own shell), then poll for the
  exit-code file, and when a wait times out, wait again. Before each launch,
  stop any earlier run still alive by its recorded ID and delete its exit-code
  file, so the poll never reads an earlier run's code. A run is finished only
  when its exit code exists (a notice that the background shell exited is not
  that); continue or end your turn only after that.
- Pre-existing uncommitted changes belong to the user. Leave them exactly as
  they are; prefer a separate worktree or branch, otherwise skip fixes that
  touch dirty files and stage your own edits by path. Edits left uncommitted by
  an interrupted earlier run of this same pass are yours: finish them through
  the targeted gate or restore their files to `HEAD`.

## 3. The pass

```
- [ ] Trust model   - [ ] Sweep   - [ ] Refute
- [ ] Fix (fix)     - [ ] Perf (if warranted)   - [ ] Final gate (fix)   - [ ] Report
```

1. **Trust model.** Read what the repo says about its own posture: what is
   internet-facing and what is internal, what is authenticated where, and
   what is fail-closed by design. Weight code over comments where they
   disagree. Done when you can state the model in a short paragraph that
   every adversary will receive.
2. **Sweep** (parallel, read-only). Spawn one `security-adversary` per
   surface, each blind to the others, each given the trust model. Match the
   count to the scope: one per surface a diff touches, 3-5 for the whole
   system. Default surfaces when the repo suggests nothing better:
   - authentication and authorization on every lane (user, service, admin,
     webhook);
   - input handling: parsing, deserialization, injection, anything arriving
     from users or third parties;
   - secrets, configuration, and logging: leakage, staleness, cache keys;
   - third-party and network boundaries: outbound calls, tenant isolation,
     SSRF;
   - data exposure in responses, errors, and artifacts.
3. **Refute** (parallel, read-only). Merge scanner and sweep findings and
   dedupe by file and issue. For each High or Critical, and each plausible
   Medium, spawn one verifier told to refute it: is it reachable given the
   deployment boundary, and does existing code already mitigate it? An
   exploit path that cannot be demonstrated means refuted. Done when every
   finding is confirmed or refuted with a reason. **Review mode ends here:**
   report the table.
4. **Fix** (fix mode, serially). Confirmed findings, High and Critical first,
   then Medium. Fix the root cause, run the targeted gate and the scanner
   that found it, and commit each fix separately (`fix(security): ...`).
   Suppress genuine scanner false positives with the tool's inline mechanism
   plus a justification comment. Record dependency vulnerabilities with no
   fix available for the report. Open security alerts on the repo host (when
   its CLI is authenticated) go through the same refute-then-fix path;
   dependency-alert fixes are targeted version bumps behind the strictest
   gate (`fix(deps): ...`).
5. **Perf** (only when warranted). Open this lane only when the arguments
   name a hotspot or the sweep found a concrete performance problem. Capture
   a baseline with the project's benchmark or the ecosystem's profiler, then
   spawn one `performance-analyst` on the capture. Apply its proposals
   serially (one `perf:` commit each), re-measure, and keep only changes whose
   number moved. Proposals flagged behavior-risky get a `behavior-verifier`
   pass. Revert on **changed**, and on any **suspect** that you cannot
   resolve to preserved with evidence.
6. **Final gate** (fix mode). Run the full check gate and every scanner again
   on the final tree. Fix or revert anything new against the baseline, and
   repeat until the ratchet holds.
7. **Report.** A findings table: finding, source (scanner / sweep / alert),
   verdict (confirmed / refuted, with reason), resolution (commit /
   suppressed with justification / deferred with owner-worthy detail). Add
   the gate and scanner commands with their before and after results, and
   benchmark numbers if the perf lane ran. Leave pushing and merging to the
   user.
