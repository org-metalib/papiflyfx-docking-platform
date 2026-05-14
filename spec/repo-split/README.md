# PapiflyFX Docking Repository Split

**Lead**: @spec-steward  
**Priority**: P2 (Normal)  
**Status**: Planning  
**Created**: 2026-05-12

## Purpose

This spec defines how to split the current `papiflyfx-docking` Maven monorepo into several repositories while preserving build integrity, module ownership, release traceability, and downstream consumer ergonomics.

The split is a repository and delivery-architecture change. It should not change public Java APIs, session formats, runtime behavior, ServiceLoader contracts, theme behavior, or sample behavior unless a later implementation plan explicitly scopes those changes.

## Spec Artifacts

- [research.md](research.md) captures the current module graph and split constraints.
- [plan.md](plan.md) defines the proposed repository boundaries and migration phases.
- [progress.md](progress.md) tracks delivery status as the split is implemented.
- [validation.md](validation.md) records validation commands, CI checks, and residual risks.

## Ownership

The split is led by `@spec-steward` because it crosses repository structure, build/release management, documentation, and downstream integration.

Required reviewers:

- `@core-architect` for `papiflyfx-docking-api`, `papiflyfx-docking-docks`, shared contracts, and restore/session invariants.
- `@ops-engineer` for Maven parent/BOM/release/CI changes, samples, settings, and archetype impact.
- `@feature-dev` for content modules, code editor modules, ServiceLoader packaging, and feature restore behavior.
- `@auth-specialist` for login, identity provider, session lifecycle, and secret-handling boundaries.
- `@qa-engineer` for split test strategy, headless TestFX setup, and integration gates.
- `@ui-ux-designer` for shared UI primitives, CSS token packaging, and visual regression risk in extracted modules.

