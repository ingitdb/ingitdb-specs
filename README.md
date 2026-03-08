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
- [Proposals](proposals/) — feature proposals and integration design studies

## Reference Implementation

- [ingitdb-cli](https://github.com/ingitdb/ingitdb-cli) — Go CLI, the reference implementation

> When working locally, both repos share the same parent directory: `../ingitdb-cli/`

## inGitDB Clients

- [github.com/ingitdb/ingitdb-cli/pkg/ingitdb](https://github.com/ingitdb/ingitdb-cli/tree/main/pkg/ingitdb) — Go package for working with local inGitDB files
- [github.com/ingitdb/dalgo2ingitdb](https://github.com/ingitdb/dalgo2ingitdb) — DALgo bridge to inGitDB
- TypeScript clients — [ingitdb-ts](https://github.com/ingitdb/ingitdb-ts) — pnpm workspace monorepo:
  - [@ingitdb/client](https://www.npmjs.com/package/@ingitdb/client) — core interface, shared types, and utilities
  - [@ingitdb/client-github](https://www.npmjs.com/package/@ingitdb/client-github) — GitHub REST API implementation
  - [@ingitdb/client-fs](https://www.npmjs.com/package/@ingitdb/client-fs) — local filesystem implementation *(coming soon)*

See [docs/components/clients.md](docs/components/clients.md) for architecture details.

## License

Apache 2.0 — see [LICENSE](LICENSE).
