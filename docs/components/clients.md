# TypeScript Clients

TypeScript client library for inGitDB — [ingitdb-ts](https://github.com/ingitdb/ingitdb-ts).

A **pnpm workspace monorepo** containing three packages.

## Packages

| Package | npm | Description |
|---------|-----|-------------|
| [`@ingitdb/client`](https://www.npmjs.com/package/@ingitdb/client) | [![npm](https://img.shields.io/npm/v/@ingitdb/client)](https://www.npmjs.com/package/@ingitdb/client) | Core interface, shared types, and utilities |
| [`@ingitdb/client-github`](https://www.npmjs.com/package/@ingitdb/client-github) | [![npm](https://img.shields.io/npm/v/@ingitdb/client-github)](https://www.npmjs.com/package/@ingitdb/client-github) | GitHub REST API implementation |
| [`@ingitdb/client-fs`](https://www.npmjs.com/package/@ingitdb/client-fs) | [![npm](https://img.shields.io/npm/v/@ingitdb/client-fs)](https://www.npmjs.com/package/@ingitdb/client-fs) | Local filesystem implementation *(coming soon)* |

## `@ingitdb/client`

Core package. Contains the transport-agnostic `IngitDbClient` interface, all shared domain types, and reusable utilities that any client implementation depends on.

**Contents:**

- `IngitDbClient` — transport-agnostic interface that all client implementations satisfy
- `Cache` — caching interface + default IndexedDB + in-memory implementation
- `CollectionSchema` — schema parsing utilities (`parseCollectionSchema`, `normalizeCollectionSchema`)
- `parseYaml` / `stringifyYaml` — YAML helpers (backed by `js-yaml`)
- `createCommittedChangesStore` — IndexedDB-backed committed changes store factory
- Shared domain types: `DatabaseConfig`, `RecordRow`, `RecordData`, `FKView`, `CollectionEntry`, `RepoMeta`, `RepoSettings`, `PendingChange`, `CommittedChangesStore`, `PendingChangesStore`

All domain types are defined in a single `src/types.ts` module to prevent circular imports between the cache, changes, and client layers.

## `@ingitdb/client-github`

GitHub REST API implementation of `IngitDbClient`. Reads and writes inGitDB data through the GitHub API, suitable for browser and server environments without local git access.

**Exports `IngitDbGithubClient`** which extends `IngitDbClient` and adds a `githubApi` field for direct access to the underlying GitHub API wrapper.

**Features:**

- Zero framework dependencies (no Vue, React, Angular, or RxJS)
- Optional GitHub token for authenticated access
- IndexedDB + in-memory caching
- Full schema support (columns, FK views, materialized views)
- Pending and committed changes tracking
- Dual ESM + CJS build with full TypeScript declarations
- 100% test coverage

**Additional dependency:** `@ingr/codec` for decoding `.ingr` materialized view binary files.

## `@ingitdb/client-fs`

Local filesystem implementation of `IngitDbClient`. Reads inGitDB data directly from a local git repository without network access.

**Status: coming soon.** The package currently exports a stub that throws `NotImplementedError` for all methods.

## Design Principles

- **Transport-agnostic interface** — `IngitDbClient` in `@ingitdb/client` defines no transport-specific fields. GitHub-specific extensions live in `@ingitdb/client-github` via `IngitDbGithubClient extends IngitDbClient`.
- **Shared types in one place** — all domain types live in `@ingitdb/client` to allow `client-github` and `client-fs` to import them without circular dependencies.
- **Workspace protocol** — `client-github` and `client-fs` declare `@ingitdb/client` as a dependency using `workspace:*` so local development uses the workspace copy.
