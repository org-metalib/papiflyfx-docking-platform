# PapiflyFX Docking Repository Split Plan

**Lead**: @spec-steward  
**Priority**: P2 (Normal)  
**Status**: Planned  

---

## Intake Summary

| Field | Value |
| --- | --- |
| Goal | Split `papiflyfx-docking` into several repositories without changing runtime behavior. |
| Lead | `@spec-steward` |
| Required reviewers | `@core-architect`, `@ops-engineer`, `@feature-dev`, `@auth-specialist`, `@qa-engineer`, `@ui-ux-designer` |
| Impacted modules | All current Maven modules, root build files, specs, repository docs, CI, release metadata |
| Priority | P2 - planned architectural delivery work |
| Key invariant | The split must preserve artifact coordinates, public APIs, ServiceLoader behavior, session/settings/login behavior, theme assets, and headless testability unless a later plan explicitly scopes a breaking change. |

## Objectives

1. Preserve current Maven artifact coordinates under `org.metalib.papifly.docking`.
2. Preserve useful Git history for each extracted repository.
3. Keep extracted repositories independently buildable and testable.
4. Keep consumers on a simple BOM-based dependency model.
5. Keep `samples` and `archetype` as downstream integration proof, not hidden reactor-only consumers.
6. Keep release order explicit and automatable.

## Non-Goals

- Do not redesign Java public APIs or SPIs as part of the repository split.
- Do not split every Maven artifact into its own repository in the first phase.
- Do not introduce compatibility shims unless an actual API change is later scoped.
- Do not move package names away from `org.metalib.papifly.fx.*`.
- Do not change session, settings, login, ribbon, or content restore formats.

## Pre-flight Requirements

Before any extraction phase runs:

- `git filter-repo` is installed (`brew install git-filter-repo` on macOS) and `git filter-repo --version` succeeds.
- The monorepo working tree is clean (`git status --porcelain` is empty).
- `./mvnw -Dtestfx.headless=true clean verify` is green on the current `HEAD`. Extraction must not start from a red baseline.
- The current `<version>` is read from the root `pom.xml` and recorded as the `papiflyfx.version` baseline. Do not trust other documents for this value.
- A baseline tag is created in the monorepo, name `pre-split-${papiflyfx.version-without-SNAPSHOT}` (for example `pre-split-0.0.25`).
- The GitNexus PostToolUse hook at `~/.claude/hooks/gitnexus/gitnexus-hook.cjs` has been verified to pass `--skip-agents-md`; otherwise the first commit in each extracted repo will rewrite its `CLAUDE.md` / `AGENTS.md`.

## Local Extraction Layout

Extracted repositories are produced under `~/github/papiflyfx/`:

```
~/github/papiflyfx/
├── .m2-split/                   # isolated local Maven repo for the split
├── papiflyfx-build-parent/
├── papiflyfx-docking-core/
├── papiflyfx-docking-settings/
├── papiflyfx-docking-login/
├── papiflyfx-docking-code/
├── papiflyfx-docking-content/
└── papiflyfx-docking-platform/
```

`.m2-split/` is a dedicated local repository used by every per-phase build via `-Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split`. This keeps split snapshots out of the developer's main `~/.m2/repository` and guarantees that downstream phases only see artifacts that were actually installed by the split workflow.

## Target Repository Boundaries

| Repository | Modules | Spec folders | Primary owner | Notes |
| --- | --- | --- | --- | --- |
| `papiflyfx-build-parent` | build parent POM only | none | `@ops-engineer` | Shared Java, JavaFX, plugin, test, release, and publishing management. No source modules. Ships an `it/smoke-consumer/` integration POM that inherits from the parent and runs `mvn -N verify` to catch plugin-config regressions in the parent itself. |
| `papiflyfx-docking-core` | `papiflyfx-docking-api`, `papiflyfx-docking-docks` | `spec/papiflyfx-docking-api/`, `spec/papiflyfx-docking-docks/` | `@core-architect` | Foundational contracts and runtime. Extract first. |
| `papiflyfx-docking-settings` | `papiflyfx-docking-settings-api`, `papiflyfx-docking-settings` | `spec/papiflyfx-docking-settings*` | `@ops-engineer` | Depends on core artifacts. |
| `papiflyfx-docking-login` | `papiflyfx-docking-login-idapi`, `papiflyfx-docking-login-session-api`, `papiflyfx-docking-login` | `spec/papiflyfx-docking-login*` | `@auth-specialist` | Depends on core and settings API. |
| `papiflyfx-docking-code` | `papiflyfx-docking-code`, `papiflyfx-docking-code-java`, `papiflyfx-docking-code-javascript`, `papiflyfx-docking-code-json`, `papiflyfx-docking-code-yaml`, `papiflyfx-docking-code-markdown` | `spec/papiflyfx-docking-code*`, `spec/papiflyfx-docking-code-lang-plugin/` | `@feature-dev` | Keep editor and language packs together initially. |
| `papiflyfx-docking-content` | `papiflyfx-docking-tree`, `papiflyfx-docking-media`, `papiflyfx-docking-hugo`, `papiflyfx-docking-github` | `spec/papiflyfx-docking-tree*`, `spec/papiflyfx-docking-media*`, `spec/papiflyfx-docking-hugo*`, `spec/papiflyfx-docking-github*` | `@feature-dev` | Can later split into per-feature repos if release cadence diverges. |
| `papiflyfx-docking-platform` | `papiflyfx-docking-bom`, `papiflyfx-docking-samples`, `papiflyfx-docking-archetype` | `AGENTS.md`, `CLAUDE.md`, `spec/agents/`, `spec/repo-split/`, `spec/papiflyfx-docking-archetype/`, and any remaining cross-cutting docs | `@ops-engineer` with `@spec-steward` | Integration and consumer-facing alignment. Extract last. Holds the shared agent team operating model; per-module spec folders move with their respective repos. |

## Versioning Strategy

Start with lock-step versions across repositories, using the current project version as the initial split baseline. Cross-repo dependencies should be managed by the BOM or a `papiflyfx.version` property, not `${project.version}`.

Independent versioning can be introduced later only after:

- CI verifies downstream repositories against released upstream artifacts.
- The BOM can describe mixed artifact versions cleanly.
- Release notes can explain compatible version ranges.

## Maven Conversion Rules

1. Each extracted repo gets its own root aggregator POM with only local modules.
2. Each extracted repo inherits from `papiflyfx-build-parent`.
3. Same-repo module dependencies may use `${project.version}`.
4. Cross-repo PapiflyFX dependencies must use BOM-managed versions or an explicit `papiflyfx.version` property; `${project.version}` is forbidden for cross-repo coordinates.
5. The BOM remains a consumer dependency-management artifact and must not include `papiflyfx-docking-samples`.
6. Generated archetype projects should import the BOM and depend on published framework artifacts.
7. Root `scm`, `url`, release profile, and publishing metadata must be corrected per repository.
8. ServiceLoader files under `META-INF/services` must remain in the provider module that owns the implementation.
9. `papiflyfx-build-parent` ships a minimal `it/smoke-consumer/` POM that inherits from the parent and runs `mvn -N verify` so plugin-management or property regressions are caught in the parent's own build.
10. Every per-phase build runs against the isolated local repository: `-Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split`. This applies to all `clean install`, `verify`, and `test` invocations during the split.
11. Per-phase install (`./mvnw clean install`), not just `test`, is the gate that promotes a repo's snapshots into `.m2-split` for downstream phases to consume.

## Extraction Workflow

Use fresh `--no-local` clones for history rewriting. Do not run history-rewriting commands in the primary working tree.

Every extraction must retain the shared build skeleton in addition to its module-specific paths. The minimum filter-repo path set for any extracted repo is:

```
--path pom.xml
--path README.md
--path LICENSE
--path .gitignore
--path .editorconfig
--path mvnw
--path mvnw.cmd
--path .mvn/
```

Phases 2–7 add their module directories and matching `spec/` subfolders on top of this base. `AGENTS.md`, `CLAUDE.md`, and `spec/agents/` stay with `papiflyfx-docking-platform`; per-repo agent guidance is generated from a trimmed copy of `AGENTS.md` containing only the roles that own modules in that repo.

Example for `papiflyfx-docking-core`:

```bash
git clone --no-local /path/to/papiflyfx-docking ~/github/papiflyfx/papiflyfx-docking-core
cd ~/github/papiflyfx/papiflyfx-docking-core

git filter-repo \
  --path pom.xml \
  --path README.md \
  --path LICENSE \
  --path .gitignore \
  --path .editorconfig \
  --path mvnw \
  --path mvnw.cmd \
  --path .mvn/ \
  --path papiflyfx-docking-api/ \
  --path papiflyfx-docking-docks/ \
  --path spec/papiflyfx-docking-api/ \
  --path spec/papiflyfx-docking-docks/
```

After filtering, for every extracted repo:

1. Rewrite the root `pom.xml` to list only local modules and to inherit from `papiflyfx-build-parent:${papiflyfx.version}` (Phases 2+).
2. Replace any cross-repo `${project.version}` dependency references with `${papiflyfx.version}`.
3. Add or update a repository-scoped `README.md` and a trimmed `AGENTS.md` containing only the agent roles whose modules live in this repo.
4. Update `<scm>`, `<url>`, and any release/publishing metadata to point at the new repository.
5. Add the per-repo `.github/workflows/ci.yml` caller (see CI Strategy below).
6. Commit with a message of the form `Extract <repo-name> from papiflyfx-docking monorepo`.
7. Tag provenance: `git tag imported-from-monorepo-<short-sha>` where `<short-sha>` is the monorepo's `HEAD` at extraction time.
8. Build and install snapshots into the isolated local repo:

   ```bash
   ./mvnw -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split \
          -Dtestfx.headless=true clean install
   ```
9. Do not push to any remote during the split. Remote creation and the first `git push` are an explicit follow-up action for the user, not part of the automated workflow.

If a `filter-repo` invocation reports unexpected references (submodules, LFS pointers, oversized blobs) the workflow stops; the clone is deleted and re-attempted from scratch with a corrected path list. `filter-repo` is never re-run with different paths on the same clone.

## CI Strategy

To avoid plugin-config drift across seven repositories, CI is structured as a single reusable workflow plus thin callers:

- `papiflyfx-build-parent` hosts a reusable workflow at `.github/workflows/papiflyfx-ci.yml` (using `workflow_call`) that performs the standard headless TestFX build (`./mvnw -Dtestfx.headless=true clean verify`) on the supported OS matrix.
- Every other repo ships a `.github/workflows/ci.yml` whose body is a small `uses: org-metalib/papiflyfx-build-parent/.github/workflows/papiflyfx-ci.yml@<ref>` caller plus repo-specific inputs (test profile, JDK version override, etc.).
- Release workflows follow the same caller/reusable pattern, defined in `papiflyfx-build-parent` and invoked from each downstream repo.

Until the GitHub remotes exist, the per-repo `ci.yml` files are committed but inert.

## Migration Phases

### Phase 0: Baseline and Freeze

- Verify `git filter-repo` is installed locally.
- Read `<version>` from the monorepo's root `pom.xml` and pin it as the working `papiflyfx.version`. Do not trust other documents for this value.
- Confirm the working tree is clean.
- Run `./mvnw -Dtestfx.headless=true clean verify` and require it to be green.
- Create the baseline tag `pre-split-${papiflyfx.version-without-SNAPSHOT}` (for example `pre-split-0.0.25`).
- Confirm the GitNexus PostToolUse hook at `~/.claude/hooks/gitnexus/gitnexus-hook.cjs` already passes `--skip-agents-md`; if not, fix the hook before any extraction commit.
- Create `~/github/papiflyfx/` and `~/github/papiflyfx/.m2-split/`.
- Pause unrelated broad refactors until extraction branches are created.

### Phase 1: Build Parent

- Extract a no-module `papiflyfx-build-parent`.
- Move shared Java, JavaFX, pluginManagement, test, release, and publishing configuration there.
- Add `papiflyfx.version` as a property in the parent, initialized to the current monorepo version.
- Add the `it/smoke-consumer/` integration POM that inherits from the parent and verifies the parent itself with `mvn -N verify`.
- Add the reusable CI workflow at `.github/workflows/papiflyfx-ci.yml`.
- Install the parent into `.m2-split` so all downstream phases can resolve it.

### Phase 2: Core Repository

- Extract `api` and `docks`.
- Preserve shared UI resources, CSS tokens, docking contracts, ribbon contracts, and restore/session behavior.
- Publish/install snapshots for downstream work.

### Phase 3: Settings Repository

- Extract `settings-api` and `settings`.
- Convert dependencies on `api` to upstream artifact dependencies.
- Validate settings runtime, settings panel tests, and ServiceLoader patterns.

### Phase 4: Login Repository

- Extract `login-idapi`, `login-session-api`, and `login`.
- Consume `settings-api` and core from upstream artifacts.
- Validate session lifecycle, provider SPI tests, and secret-handling paths.

### Phase 5: Code Repository

- Extract code editor and built-in language modules.
- Validate language provider ServiceLoader registration, editor state restore, and benchmark exclusion behavior.

### Phase 6: Content Repository

- Extract tree, media, hugo, and github modules.
- Validate feature content factories, content state adapters, ribbon providers, and module CSS packaging.

### Phase 7: Platform Repository

- Extract BOM, samples, archetype, and cross-repo docs.
- Convert samples and archetype to consume published or locally installed artifacts.
- Use this repo as the integration proof for the split.

### Phase 8: Monorepo Retirement

- Keep the old monorepo archived or convert it into a pointer repository.
- Document the new repository map and release process.
- Update project metadata, issue templates, and developer onboarding docs.

## Validation Strategy

All split-time builds run against the isolated local repository:

```bash
export PAPIFLYFX_M2=$HOME/github/papiflyfx/.m2-split
```

Per repository (gate before moving to the next phase):

```bash
./mvnw -Dmaven.repo.local=$PAPIFLYFX_M2 -Dtestfx.headless=true clean install
```

`install` (not bare `test`) is the gate: it both runs the tests and populates `.m2-split` for downstream phases.

Platform integration after Phases 1–6 have all installed cleanly:

```bash
./mvnw -Dmaven.repo.local=$PAPIFLYFX_M2 \
       -pl papiflyfx-docking-samples -am \
       -Dtest=SamplesSmokeTest -Dtestfx.headless=true verify

./mvnw -Dmaven.repo.local=$PAPIFLYFX_M2 \
       -pl papiflyfx-docking-archetype -am verify
```

The archetype's integration tests are the strongest end-to-end signal because they generate a project from the BOM and consume every framework artifact through ordinary Maven resolution.

Release-order validation:

1. Install `papiflyfx-build-parent` into `.m2-split`.
2. Install core.
3. Install settings.
4. Install login.
5. Install code.
6. Install content.
7. Build platform from a clean state using only artifacts present in `.m2-split` (no fallback to `~/.m2/repository` or remote mirrors during the split rehearsal).

## Acceptance Criteria

- Each target repository builds independently with `./mvnw -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split -Dtestfx.headless=true clean install`.
- Each target repository has a scoped `README.md`, a trimmed `AGENTS.md`, a `.github/workflows/ci.yml` caller, updated `<scm>` / `<url>`, and an `imported-from-monorepo-<sha>` provenance tag on its initial commit.
- `papiflyfx-build-parent` ships an `it/smoke-consumer/` POM that succeeds with `mvn -N verify` against the parent.
- The platform repo builds `samples` and `archetype` against `.m2-split` only, without touching `~/.m2/repository` or remote mirrors.
- The BOM manages all published framework artifacts and excludes `samples`.
- All ServiceLoader-based providers (language packs, syntax styles, content state adapters, identity providers) are discoverable after extraction.
- Headless TestFX configuration works in every repo with UI tests.
- No public Java API or persistence format changes are introduced by the split.
- The monorepo has a `pre-split-<version>` tag at the extraction baseline; the old monorepo is either archived or documented as superseded.

