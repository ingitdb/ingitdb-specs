# ingitdb-specs

Specifications, roadmap, and design documentation for [inGitDB](https://ingitdb.com) — covering the storage format, schema definitions, system-level features, and component behaviors shared across all implementations.

## Contents

- [Storage Format](docs/STORAGE_FORMAT.md) — directory layout, config files, collection schema, column types, record files
- [Schema Reference](docs/schema/) — `.definition.yaml`, views, triggers, subcollections, subscribers
- [Features](docs/features/) — feature specifications with status (implemented / WIP / planned)
- [Components](docs/components/) — Validator, Scanner, Views Builder, Watcher, Triggers, MCP Server, Merge Conflict Resolver, Migration Generator
- [Roadmap](docs/ROADMAP.md) — nine delivery phases from Validator to GraphQL
- [Backlog](docs/BACKLOG.md) — concrete tasks with acceptance criteria, ordered by dependency
- [Competitors](docs/COMPETITORS.md) — honest feature comparison with related tools
- [Guidelines](docs/GUIDELINES.md) — design and process guidelines
- [GitHub App](docs/github-app/) — plans for dependency validation and PR auto-merge GitHub App
- [Plans](plans/) — migration and design decision records

## Reference Implementation

- [ingitdb-cli](https://github.com/ingitdb/ingitdb-cli) — Go CLI, the reference implementation

> When working locally, both repos share the same parent directory: `../ingitdb-cli/`

## License

Apache 2.0 — see [LICENSE](LICENSE).
