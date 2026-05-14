# Repository Split Progress

## Current Status

Planning refined. No build files or source modules have been changed yet. The execution prompt at `execution-prompt.md` is ready for an implementation agent.

## Completed

- [x] Created `spec/repo-split/` as the canonical planning location.
- [x] Captured initial module graph and split constraints in `research.md`.
- [x] Drafted repository boundaries, migration phases, and acceptance criteria in `plan.md`.
- [x] Filled review gaps: pre-flight checklist, full filter-repo path list, `spec/` ownership per repo, build-parent smoke consumer, isolated `.m2-split` local Maven repo, install-as-gate rule, GitNexus hook caveat, repository naming conflict note, CI reusable-workflow strategy, baseline and provenance tag conventions.
- [x] Produced `execution-prompt.md` so a fresh agent session can execute the split locally with `~/github/papiflyfx/` as the output root.

## Pending

- [ ] Decide whether to rename umbrella repositories (`papiflyfx-docking-settings` vs `papiflyfx-settings`, etc.).
- [ ] Review plan with `@core-architect`.
- [ ] Review build/release strategy with `@ops-engineer`.
- [ ] Review content/code extraction scope with `@feature-dev`.
- [ ] Review login/session/security boundaries with `@auth-specialist`.
- [ ] Review headless test and CI plan with `@qa-engineer`.
- [ ] Review shared UI asset packaging risk with `@ui-ux-designer`.
- [ ] Confirm GitNexus PostToolUse hook passes `--skip-agents-md`.
- [ ] Create baseline tag `pre-split-<version>` before extraction work.
- [ ] Extract `papiflyfx-build-parent`.
- [ ] Extract `papiflyfx-docking-core`.
- [ ] Extract downstream repositories in dependency order.
- [ ] Convert platform repo into the cross-repo integration gate.

## Handoff

Lead Agent: `@spec-steward`  
Task Scope: Repository split planning only.  
Impacted Modules: All current modules are in scope for future extraction.  
Files Changed: `spec/repo-split/**`.  
Key Invariants: No source behavior, public APIs, Maven artifact coordinates, ServiceLoader contracts, session formats, settings formats, login flows, or UI resource packaging should change during the split unless explicitly planned later.  
Validation Performed: Documentation review and gap-fill pass against an independent review.  
Open Risks / Follow-ups: Repository naming decision (collision between `papiflyfx-docking-settings` repo and module), reusable CI workflow contents, archetype IT against `.m2-split` only.  
Required Reviewer: `@ops-engineer` should review first because the next phase is build-parent extraction; `@spec-steward` should confirm the naming-collision decision before Phase 3.

