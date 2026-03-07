# Plan 001: Migrate spec-level docs from ingitdb-cli to ingitdb-specs

## Context

`ingitdb-cli` accumulated ~145 markdown files covering both CLI-specific implementation details and broader inGitDB format/behavior specifications. As the project grows to multiple implementations (ingitdb-ts, ingitdb-ws, etc.), spec-level content needs a canonical home in `ingitdb-specs` so all implementations can reference it without duplicating or diverging.

## What moved

### From `ingitdb-cli/docs/` → `ingitdb-specs/docs/`

| Source | Destination |
|--------|-------------|
| `ROADMAP.md` | `docs/ROADMAP.md` |
| `BACKLOG.md` | `docs/BACKLOG.md` |
| `COMPETITORS.md` | `docs/COMPETITORS.md` |
| `GUIDELINES.md` | `docs/GUIDELINES.md` |
| `schema/` (all files) | `docs/schema/` |
| `features/` (all files) | `docs/features/` |
| `components/README.md` | `docs/components/README.md` |
| `components/validator/` | `docs/components/validator/` |
| `components/scanner.md` | `docs/components/scanner.md` |
| `components/views-builder.md` | `docs/components/views-builder.md` |
| `components/watcher.md` | `docs/components/watcher.md` |
| `components/triggers.md` | `docs/components/triggers.md` |
| `components/mcp-server.md` | `docs/components/mcp-server.md` |
| `components/merge-conflict-resolver.md` | `docs/components/merge-conflict-resolver.md` |
| `components/migration-generator.md` | `docs/components/migration-generator.md` |
| `github-app/` (all files) | `docs/github-app/` |

### Split: `ingitdb-cli/docs/ARCHITECTURE.md`

- Data model / storage format section → `ingitdb-specs/docs/STORAGE_FORMAT.md` (new file)
- CLI component architecture section → remains as `ingitdb-cli/docs/ARCHITECTURE.md` (trimmed)

## What stayed in ingitdb-cli

- `docs/cli/` — CLI command reference
- `docs/configuration/` — CLI user and repository config
- `docs/components/readme-builders/` — CLI-specific output component
- `docs/CONTRIBUTING.md`, `docs/CODING_STANDARDS.md`, `docs/CI.md` — CLI contributor docs
- `docs/release/` — packaging and release procedures
- `macos-notarization.md`, `windows-signing.md`, `RELEASE_SETUP.md`
- `DALGO2GHINGITDB.md` — DALgo integration (CLI-specific)
- `cmd/ingitdb/commands/IMPLEMENTATION_SUMMARY.md`, `TESTING.md`

## Cross-referencing

- Both repos are on GitHub under the `ingitdb` org.
- `ingitdb-cli` README and docs index link to `https://github.com/ingitdb/ingitdb-specs` for spec-level content.
- `ingitdb-specs` README links back to `https://github.com/ingitdb/ingitdb-cli`.
- Locally, repos share the parent directory `/projects/ingitdb/`.
