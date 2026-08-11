---
name: make-it-right
description: Phase 2 refactoring pass over a branch's changes, orchestrated with subagents - scouts map the changed surface, refactor-proposer agents propose per dimension, changes apply serially behind the project's check gate, and behavior-verifier agents attack the result. Use when a feature works but hasn't had a cleanup pass, or when asked to "make it right", clean up, or refactor a branch. Never changes behavior.
---

Orchestrate a behavior-preserving cleanup of this branch. Subagents propose in
parallel (read-only); you apply serially, one concern per commit, with the
project's check gate after every commit; adversarial verifiers attack the
result before you report success.

Harness note: role playbooks live in `roles/` next to this file. If your
harness supports named subagents, register the roles and spawn them by name in
parallel. If it supports plain subagents, paste each role playbook into the
subagent's instructions. If it has no subagents at all, perform each role
yourself, sequentially. The stages and gates below stay identical either way.

0. **Find the check gate.** The gate is the project's canonical verification
   command — ideally one verb covering lint + typecheck + tests. Look for it in
   the repo's agent/contributor docs (AGENTS.md, CLAUDE.md, CONTRIBUTING,
   README) and task-runner config (Makefile, justfile, mise.toml, package.json
   scripts, or the ecosystem's equivalent). If no single command exists,
   compose one from the individual lint/typecheck/test commands and use that
   composition consistently. If nothing runnable exists, stop and report —
   never refactor without a gate.

1. **Scope and baseline.** Changed files: diff against the merge-base with the
   main branch (or the base/paths given as arguments to this skill). Drop
   vendored, generated, and lockfile paths. Run the check gate. If red, stop
   and report — never refactor on a red baseline. If the diff is empty, stop
   and say so.

2. **Scout (parallel, read-only).** Run 2 read-only scout subagents:
   - one hunting existing helpers, utilities, and types the changed code
     should have reused (search sibling modules and shared/library code);
   - one mapping conventions the changed code deviates from (compare with 2-3
     neighboring modules of the same kind).

3. **Propose (parallel, read-only).** Run 4 subagents with the
   refactor-proposer role (`roles/refactor-proposer.md`), one per dimension:
   `naming`, `duplication`, `structure-coupling`, `idioms-and-concurrency`.
   Give each the changed-file list, the branch goal (from recent commit
   messages), and the relevant scout findings. Each returns ranked proposals
   with file:line anchors — no edits.

4. **Synthesize.** Merge the proposal lists. Drop anything that would change
   observable behavior, public API, route paths, serialized field names, or
   test assertions; drop new dependencies; drop "wrong abstraction" dedupes
   (instances that only look similar). Where proposals conflict, prefer the
   smaller change. Order the survivors: renames → dedupes → moves/splits —
   renames first so extracted helpers get good names.

5. **Apply serially.** For each surviving proposal:
   - make the edit(s), updating every reference (search, don't trust memory);
   - run the check gate;
   - green → commit (`refactor: rename X to Y` / `refactor: extract X` /
     `refactor: move X to Y`) — one refactoring kind per commit;
   - red → revert the working tree, retry a smaller version once, else skip
     and record why.
   Never batch unrelated proposals into one commit.

6. **Verify (parallel, adversarial).** Run 2 subagents with the
   behavior-verifier role (`roles/behavior-verifier.md`) over the new commit
   range, telling each to attack it independently. Any **changed** verdict:
   fix or revert that commit, re-run the check gate, and re-verify. A
   **suspect** verdict: investigate yourself; resolve to
   preserved-with-evidence or revert.

7. **Report.** Final message must include: commits made (hash + subject),
   proposals rejected in synthesis and why, proposals skipped during apply and
   why, verifier verdicts, and anything needing human judgment (public API
   changes, cross-package moves, suspicious-but-unproven behavior
   differences). Do not push or merge — leave the branch for review.
