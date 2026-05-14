# Repository Split Research

## Current State

The canonical source for the current project version is the root `pom.xml` `<version>` element. Other documents (notably `CLAUDE.md`) may quote stale values; the split must read the version from the POM at extraction time and pin it as `papiflyfx.version` across the new repositories.

The current repository is a single Maven aggregator with these modules:

- `papiflyfx-docking-bom`
- `papiflyfx-docking-api`
- `papiflyfx-docking-settings-api`
- `papiflyfx-docking-settings`
- `papiflyfx-docking-docks`
- `papiflyfx-docking-code`
- `papiflyfx-docking-code-java`
- `papiflyfx-docking-code-javascript`
- `papiflyfx-docking-code-json`
- `papiflyfx-docking-code-yaml`
- `papiflyfx-docking-code-markdown`
- `papiflyfx-docking-tree`
- `papiflyfx-docking-media`
- `papiflyfx-docking-hugo`
- `papiflyfx-docking-github`
- `papiflyfx-docking-login-idapi`
- `papiflyfx-docking-login-session-api`
- `papiflyfx-docking-login`
- `papiflyfx-docking-samples`
- `papiflyfx-docking-archetype`

The root POM centralizes Java, JavaFX, dependency management, plugin management, headless test defaults, and release/publishing profiles. Splitting repositories therefore requires extracting shared build configuration before downstream repositories can build reliably outside the reactor.

## Internal Dependency Graph

The current internal Maven dependency graph has no obvious runtime cycles.

| Module | Internal dependencies |
| --- | --- |
| `papiflyfx-docking-api` | none |
| `papiflyfx-docking-docks` | `papiflyfx-docking-api` |
| `papiflyfx-docking-settings-api` | `papiflyfx-docking-api` |
| `papiflyfx-docking-settings` | `papiflyfx-docking-settings-api`, `papiflyfx-docking-api`, `papiflyfx-docking-docks` test-scope |
| `papiflyfx-docking-login-idapi` | none |
| `papiflyfx-docking-login-session-api` | `papiflyfx-docking-login-idapi`, `papiflyfx-docking-settings-api` |
| `papiflyfx-docking-login` | `papiflyfx-docking-login-idapi`, `papiflyfx-docking-login-session-api`, `papiflyfx-docking-api`, `papiflyfx-docking-settings-api`, `papiflyfx-docking-docks` test-scope |
| `papiflyfx-docking-code` | `papiflyfx-docking-api`, `papiflyfx-docking-settings-api`, `papiflyfx-docking-docks` test-scope |
| `papiflyfx-docking-code-*` language modules | `papiflyfx-docking-code` |
| `papiflyfx-docking-tree` | `papiflyfx-docking-api`, `papiflyfx-docking-docks` test-scope |
| `papiflyfx-docking-media` | `papiflyfx-docking-api`, `papiflyfx-docking-docks` test-scope |
| `papiflyfx-docking-hugo` | `papiflyfx-docking-api`, `papiflyfx-docking-settings-api`, `papiflyfx-docking-docks` test-scope |
| `papiflyfx-docking-github` | `papiflyfx-docking-api`, `papiflyfx-docking-settings-api`, `papiflyfx-docking-docks` test-scope |
| `papiflyfx-docking-samples` | all primary runtime/content modules |
| `papiflyfx-docking-bom` | dependency management for published framework modules |
| `papiflyfx-docking-archetype` | generated-project templates and integration test resources |

## Split Constraints

- `papiflyfx-docking-api` must remain the lowest-level docking and shared UI contract artifact.
- `papiflyfx-docking-docks` should initially stay with `api` because it is the core runtime over those contracts.
- Modules with only test-scope dependency on `docks` can move to separate repos once `docks` is published and available to tests.
- `settings-api` depends on `api`; `settings` should stay with `settings-api` unless separate third-party settings runtimes become a real release need.
- `login-session-api` depends on both `login-idapi` and `settings-api`; the login repository must consume the settings API from an upstream artifact.
- The code editor language modules depend on `papiflyfx-docking-code`; they should move together first and can split later if language packs need independent release cadence.
- `samples` is the broadest integration consumer and should be moved last.
- The BOM should exclude `samples` and remain a consumer-facing alignment artifact.
- The archetype should consume published artifacts rather than relying on a local reactor.

## Key Risks

- Cross-repo snapshots can hide broken release ordering if CI only builds individual repositories.
- Reusing `${project.version}` for cross-repo dependencies will be wrong after extraction.
- Duplicating root POM plugin configuration in each repo will create drift unless a shared build parent is introduced.
- ServiceLoader resources must remain packaged with their provider modules after extraction.
- TestFX, Monocle, JavaFX classifier profiles, and native-access flags need to stay consistent in every extracted repo.
- Git history rewriting must happen in fresh clones, not in the primary working tree.
- The current repository-level docs and agent instructions will need per-repo scoping after extraction.
- Extraction filter lists must include the Maven wrapper (`mvnw`, `mvnw.cmd`, `.mvn/`), `.gitignore`, `.editorconfig`, `LICENSE`, and any module-local `META-INF/services/` descriptors. Omitting these makes the extracted repo unbuildable or silently drops ServiceLoader providers.
- The local GitNexus PostToolUse hook (`~/.claude/hooks/gitnexus/gitnexus-hook.cjs`) regenerates `CLAUDE.md` / `AGENTS.md` on commit and merge. Unless it already passes `--skip-agents-md`, the first commit in each extracted repo will overwrite the per-repo agent documentation produced during the split.

## Repository Naming Note

The target repository `papiflyfx-docking-settings` umbrellas a Maven module that is also called `papiflyfx-docking-settings`. The repo-name / module-name collision is fine on disk and in Maven coordinates, but it complicates GitHub search and grep-style audits. Two options for the implementation phase:

- Accept the collision and document it explicitly in each repo's `README.md`.
- Rename umbrella repositories to drop the `-docking-` infix (for example `papiflyfx-settings`, `papiflyfx-login`, `papiflyfx-code`, `papiflyfx-content`, `papiflyfx-platform`). Maven artifact ids stay unchanged; only the GitHub repo names move.

A decision is required before Phase 3.

