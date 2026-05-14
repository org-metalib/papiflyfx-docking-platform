# Repository Split Validation

## Initial Validation

No source or build behavior has been changed. This initial spec-only change requires documentation review, not a Maven build.

## Commands Run

- Read existing agent routing guidance from `spec/agents/README.md` and `spec/agents/spec-steward.md`.
- Inspected root Maven module list from `pom.xml`.
- Inspected internal Maven dependencies across module POMs.
- Inspected existing spec plan style from `spec/papiflyfx-docking-archetype/plan.md` and `spec/papiflyfx-docking-code-lang-plugin/plan.md`.

## Required Future Validation

### Pre-flight (before any extraction)

```bash
git filter-repo --version
./mvnw --version
java --version                                                   # must be 25
git -C <monorepo> status --porcelain                             # must be empty
./mvnw -Dtestfx.headless=true clean verify                       # must be green
git -C <monorepo> tag -a pre-split-<version> -m "Baseline before split"
mkdir -p ~/github/papiflyfx ~/github/papiflyfx/.m2-split
```

### Per extracted repository

Every per-phase build uses the isolated split-local repository so downstream phases cannot accidentally resolve artifacts from the developer's main `~/.m2/repository`:

```bash
./mvnw -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split \
       -Dtestfx.headless=true clean install
```

`install` (not `test`) is the promotion gate: it both runs tests and publishes the repo's snapshots into `.m2-split` for the next phase.

### Platform integration

The platform repository's tests are the only ones that require every upstream phase to have already installed cleanly. They must pass against `.m2-split` only:

```bash
./mvnw -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split \
       -pl papiflyfx-docking-samples -am \
       -Dtest=SamplesSmokeTest -Dtestfx.headless=true verify

./mvnw -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split \
       -pl papiflyfx-docking-archetype -am verify
```

### Provenance and rollback

- Monorepo has the baseline tag `pre-split-<version>`.
- Every extracted repo has at least one commit plus an `imported-from-monorepo-<short-sha>` tag.
- Reverting the split is "delete the per-repo clones and `.m2-split`"; the monorepo's `pre-split-<version>` tag is the last green point.

### Full consumer validation

- Generate a new app from the archetype using only `.m2-split`.
- Import the split BOM.
- Add representative dependencies from core, settings, login, code, and content.
- Verify startup, session restore, settings restore, ribbon providers, and ServiceLoader discovery.

## Residual Risks

- Cross-repo CI and release automation are not implemented yet; the plan now specifies a reusable workflow in `papiflyfx-build-parent` plus thin callers, but those files do not exist until Phase 1.
- Build-parent extraction may reveal root POM assumptions that are currently hidden by the reactor; the `it/smoke-consumer/` integration POM is the early-warning probe for that.
- Test-scope dependencies on `papiflyfx-docking-docks` require locally installed core artifacts before downstream tests can run; the `.m2-split` workflow enforces this ordering.
- `samples` may need explicit dependency versions once it stops being a reactor module; in the split it consumes `${papiflyfx.version}` via the BOM.
- The GitNexus PostToolUse hook regenerates `CLAUDE.md` / `AGENTS.md` on commit unless it passes `--skip-agents-md`. Each extracted repo's first commit is the failure window.
- The repository name `papiflyfx-docking-settings` collides with the module of the same name inside it; a naming decision is required before Phase 3 (keep with documentation, or rename umbrella repos to `papiflyfx-settings`, `papiflyfx-login`, etc.).
- `CLAUDE.md` may quote a stale project version. The POM is the canonical source — extraction reads `papiflyfx.version` from `pom.xml`, not from documentation.

