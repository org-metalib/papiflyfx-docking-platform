# Repository Split — Execution Prompt

> Paste this entire file as the opening message to a fresh Claude Code (or Cowork) session that has shell + file access to the monorepo and write access to `~/github/papiflyfx/`. Then approve the plan it produces before letting it run any history-rewriting commands.

---

## Mission

You are executing the PapiflyFX Docking repository split defined in `spec/repo-split/` of the monorepo at:

```
/Users/igor/github/ikatraev/mixhawkmusic-ws/github/papiflyfx-docking/target/papiflyfx-docking
```

Produce seven independent Git repositories under `~/github/papiflyfx/` (absolute: `/Users/igor/github/papiflyfx/`), each independently buildable with `./mvnw -Dtestfx.headless=true verify`, with cross-repo dependencies resolved through a locally installed snapshot at `~/github/papiflyfx/.m2-split`.

**Target repositories** (in extraction order):

1. `papiflyfx-build-parent` — no-module parent POM
2. `papiflyfx-docking-core` — `papiflyfx-docking-api`, `papiflyfx-docking-docks`
3. `papiflyfx-docking-settings` — `papiflyfx-docking-settings-api`, `papiflyfx-docking-settings`
4. `papiflyfx-docking-login` — `papiflyfx-docking-login-idapi`, `papiflyfx-docking-login-session-api`, `papiflyfx-docking-login`
5. `papiflyfx-docking-code` — `papiflyfx-docking-code` + the five language packs (`-java`, `-javascript`, `-json`, `-yaml`, `-markdown`)
6. `papiflyfx-docking-content` — `papiflyfx-docking-tree`, `papiflyfx-docking-media`, `papiflyfx-docking-hugo`, `papiflyfx-docking-github`
7. `papiflyfx-docking-platform` — `papiflyfx-docking-bom`, `papiflyfx-docking-samples`, `papiflyfx-docking-archetype`, cross-repo `spec/`, `AGENTS.md`, `spec/agents/`, `spec/repo-split/`

## Invariants — DO NOT VIOLATE

- Do not modify Java source, public APIs, ServiceLoader files, session/settings/login formats, or theme assets.
- Do not change Maven `groupId` (`org.metalib.papifly.docking`) or `artifactId` for any module.
- Do not change package names (`org.metalib.papifly.fx.*`).
- Do not run `git filter-repo` or any history-rewriting command in the original working tree at `target/papiflyfx-docking`. Always operate on `--no-local` clones.
- Do not use `${project.version}` for cross-repo dependencies. Use a `papiflyfx.version` property managed by the BOM.
- Do not push to any remote. This is a fully local extraction; the user will create GitHub remotes and push manually later.
- If any phase's validation fails, **stop and report**. Do not auto-fix and continue.

## Pre-flight (Phase 0)

Before any extraction:

1. Verify tooling:
   ```bash
   git --version
   git filter-repo --version || echo "INSTALL REQUIRED: brew install git-filter-repo"
   ./mvnw --version
   java --version   # must be 25
   ```
2. Confirm the monorepo is clean: `git -C <monorepo> status --porcelain` returns empty.
3. Read and record the current version from the root `pom.xml` `<version>` element. Save it as the `PAPIFLY_VERSION` variable used throughout the run. Use the literal value from the POM (do not assume `0.0.15`; the actual current value is `0.0.25-SNAPSHOT` or later — read it).
4. In the monorepo, create the baseline tag (do not push):
   ```bash
   git tag -a pre-split-${PAPIFLY_VERSION%-SNAPSHOT} -m "Baseline before repository split"
   ```
5. Run the full reactor build and tests once to confirm green baseline:
   ```bash
   ./mvnw -Dtestfx.headless=true clean verify
   ```
   If this fails, stop. Do not extract from a red baseline.
6. Confirm the GitNexus PostToolUse hook (`~/.claude/hooks/gitnexus/gitnexus-hook.cjs`) already passes `--skip-agents-md`. If not, warn the user — the hook will overwrite `CLAUDE.md`/`AGENTS.md` in each extracted repo on its first commit. Do not silently modify the hook.
7. Prepare the output area:
   ```bash
   mkdir -p ~/github/papiflyfx
   mkdir -p ~/github/papiflyfx/.m2-split
   ```
   All subsequent Maven invocations in extracted repos must use:
   ```bash
   -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split
   ```
   This keeps the split's snapshots out of your main `~/.m2/repository`.

## Per-Phase Workflow

For every extracted repo:

1. `git clone --no-local <monorepo> ~/github/papiflyfx/<repo-name>`
2. `cd ~/github/papiflyfx/<repo-name>`
3. Run `git filter-repo --path ...` with the per-phase path list (see each phase below). Include `mvnw`, `mvnw.cmd`, `.mvn/`, `.gitignore`, `.editorconfig`, `LICENSE`, plus phase-specific paths.
4. Rewrite the root `pom.xml`: trim `<modules>` to only the modules present locally, remove the cross-repo `<scm>` and update `<url>`, set `<parent>` to `papiflyfx-build-parent` once that exists (Phase 1+).
5. Replace cross-repo Maven dependencies that previously used `${project.version}` with `${papiflyfx.version}` and ensure `papiflyfx.version` is defined either in the BOM or as a property inherited from `papiflyfx-build-parent`.
6. Add a `<dependencyManagement>` import of `papiflyfx-docking-bom` once Phase 7 has been bootstrapped — but during initial extraction, set the property directly.
7. Write a scoped `README.md` and `AGENTS.md` (carry forward only the agent roles that own modules in this repo).
8. Commit: `git add -A && git commit -m "Extract <repo-name> from papiflyfx-docking monorepo"`
9. Tag provenance: `git tag imported-from-monorepo-<short-sha-of-monorepo-HEAD>`
10. Validate:
    ```bash
    ./mvnw -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split -Dtestfx.headless=true clean install
    ```
11. If install passes, the artifacts are now available to downstream phases. Proceed.

## Phase 1 — `papiflyfx-build-parent`

Path retention list:
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

Then:
- Strip `<modules>` entirely from the root POM (this repo has no children).
- Change `<artifactId>` to `papiflyfx-build-parent` and `<packaging>` stays `pom`.
- Keep all `<properties>`, `<dependencyManagement>`, `<pluginManagement>`, profiles, release config, and headless-test defaults.
- Add a `papiflyfx.version` property initialized to the current `${PAPIFLY_VERSION}`.
- Write a minimal `it/smoke-consumer/pom.xml` that inherits from the parent and runs `mvn -N verify` — this catches plugin-config regressions in the parent itself.
- Validate: `./mvnw -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split clean install` (no tests; nothing to test yet).

## Phase 2 — `papiflyfx-docking-core`

Path retention list adds:
```
--path papiflyfx-docking-api/
--path papiflyfx-docking-docks/
--path spec/papiflyfx-docking-api/
--path spec/papiflyfx-docking-docks/
```

POM changes:
- Root `<parent>` points at `papiflyfx-build-parent:${PAPIFLY_VERSION}`.
- `<modules>` lists only `papiflyfx-docking-api` and `papiflyfx-docking-docks`.
- Inside `papiflyfx-docking-docks/pom.xml`, the dependency on `papiflyfx-docking-api` continues to use `${project.version}` (same repo — OK).
- Validate: full build + tests headless.

## Phase 3 — `papiflyfx-docking-settings`

Path retention list adds `papiflyfx-docking-settings-api/`, `papiflyfx-docking-settings/`, and matching `spec/` subfolders.

POM changes:
- Replace `papiflyfx-docking-api` and `papiflyfx-docking-docks` (test-scope) dependency versions with `${papiflyfx.version}`.
- Verify ServiceLoader files in `papiflyfx-docking-settings/src/main/resources/META-INF/services/` survived the filter.
- Validate.

## Phase 4 — `papiflyfx-docking-login`

Path retention adds `papiflyfx-docking-login-idapi/`, `papiflyfx-docking-login-session-api/`, `papiflyfx-docking-login/`, plus specs.

POM changes:
- Cross-repo deps on `settings-api`, `api`, `docks` use `${papiflyfx.version}`.
- Validate session lifecycle, provider SPI tests, secret-handling tests.

## Phase 5 — `papiflyfx-docking-code`

Path retention adds `papiflyfx-docking-code/`, `papiflyfx-docking-code-java/`, `papiflyfx-docking-code-javascript/`, `papiflyfx-docking-code-json/`, `papiflyfx-docking-code-yaml/`, `papiflyfx-docking-code-markdown/`, plus specs (`spec/papiflyfx-docking-code-lang-plugin/` etc.).

POM changes:
- Cross-repo deps use `${papiflyfx.version}`.
- Confirm Surefire `excludedGroups=benchmark` is preserved in `papiflyfx-docking-code/pom.xml`.
- Confirm each language pack's `META-INF/services/org.metalib.papifly.fx.code.spi.LanguageSupportProvider` (and `SyntaxStyleProvider`) descriptor is present after the filter.
- Validate.

## Phase 6 — `papiflyfx-docking-content`

Path retention adds `papiflyfx-docking-tree/`, `papiflyfx-docking-media/`, `papiflyfx-docking-hugo/`, `papiflyfx-docking-github/`, plus specs.

POM changes:
- Cross-repo deps use `${papiflyfx.version}`.
- Verify JGit dependency in `papiflyfx-docking-github` is unaffected.
- Validate.

## Phase 7 — `papiflyfx-docking-platform`

Path retention adds `papiflyfx-docking-bom/`, `papiflyfx-docking-samples/`, `papiflyfx-docking-archetype/`, `AGENTS.md`, `CLAUDE.md`, `spec/agents/`, `spec/repo-split/`, `spec/papiflyfx-docking-archetype/`, plus any remaining cross-cutting docs.

POM changes:
- BOM's `<dependencyManagement>` lists every upstream PapiflyFX artifact pinned to `${papiflyfx.version}` and **must not** include `papiflyfx-docking-samples`.
- `papiflyfx-docking-samples` and `papiflyfx-docking-archetype` import the BOM and use `${papiflyfx.version}`.
- Validate end-to-end:
  ```bash
  ./mvnw -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split \
         -pl papiflyfx-docking-samples -am \
         -Dtest=SamplesSmokeTest -Dtestfx.headless=true verify
  ./mvnw -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split \
         -pl papiflyfx-docking-archetype -am verify
  ```
  Both must be green before declaring the split complete.

## Post-Split Report

After Phase 7, produce a single report containing:

- Tree summary: `ls -la ~/github/papiflyfx/` plus the head commit + tag of each repo.
- The exact `papiflyfx.version` used.
- Which repos passed `clean install` and which (if any) skipped tests and why.
- A diff of `pom.xml` `<dependencyManagement>` from the original monorepo vs the new BOM, to prove no artifact was dropped.
- A checklist mapping each acceptance criterion in `spec/repo-split/plan.md` §Acceptance Criteria to a verified outcome.
- Outstanding TODOs (e.g., GitHub remotes, CI workflow files, GitNexus hook updates) for the user to action manually.

## Asking for Help

- If `git filter-repo` reports unexpected references (submodules, LFS pointers, large blobs), stop and report — do not use `--force`.
- If any phase's `mvn install` fails, stop, copy the relevant Surefire/Failsafe report lines, and ask the user how to proceed. Common likely causes: a `${project.version}` that was missed, a `META-INF/services` file dropped by the filter, or a missing `--add-opens` / `--enable-native-access` flag in a child pom that previously inherited from the monorepo root.
- Never re-run a `filter-repo` command with different paths on the same clone; delete the clone and start over.

## Definition of Done

All seven directories under `~/github/papiflyfx/` exist, each is a valid Git repository with at least one commit and an `imported-from-monorepo-*` tag, each builds green with `./mvnw -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split -Dtestfx.headless=true clean verify`, and the platform repo's samples and archetype tests pass against the locally installed snapshots only.
