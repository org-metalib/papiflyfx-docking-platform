# papiflyfx-docking-platform

Extracted from the PapiflyFX Docking monorepo.

## Modules

- `papiflyfx-docking-bom`
- `papiflyfx-docking-samples`
- `papiflyfx-docking-archetype`

## Build

Use the split-local Maven repository so cross-repo snapshots resolve from the extraction workspace:

```bash
./mvnw -Dmaven.repo.local=$HOME/github/papiflyfx/.m2-split -Dtestfx.headless=true clean verify
```

Lead agent: `@ops-engineer`.

## Notes

- The BOM does not include `papiflyfx-docking-samples`; samples consume upstream artifacts through the BOM.
