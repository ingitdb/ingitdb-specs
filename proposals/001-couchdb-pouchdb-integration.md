# Proposal 001: CouchDB/PouchDB Integration

## Overview

This proposal evaluates two integration strategies between inGitDB and
[CouchDB](https://couchdb.apache.org/) / [PouchDB](https://pouchdb.com/), and recommends
a concrete implementation path.

---

## Background

### CouchDB

[CouchDB](https://couchdb.apache.org/) is an Apache-licensed document-oriented database
that stores JSON documents and exposes a well-documented HTTP REST API. Its key properties:

- Documents stored as JSON with `_id` and `_rev` (revision) fields.
- HTTP REST API: `GET/PUT/DELETE /{db}/{docId}`, `GET /{db}/_all_docs`, `POST /{db}/_bulk_docs`.
- A **changes feed** (`GET /{db}/_changes`) that streams document mutations in sequence order.
- A deterministic **replication protocol** built on top of the changes feed, enabling
  master-master sync between any two CouchDB-compatible endpoints.
- MapReduce views stored as `_design` documents.

### PouchDB

[PouchDB](https://pouchdb.com/) is a JavaScript database library (browser + Node.js)
that implements the CouchDB replication protocol locally. Key properties:

- Runs fully offline in the browser (IndexedDB) or in Node.js.
- Can sync bidirectionally with any CouchDB-compatible HTTP endpoint.
- Widely used in offline-first web and mobile apps.
- The sync target can be a real CouchDB server, or any server that implements the
  CouchDB replication protocol.

---

## Approaches Considered

### Approach A — CouchDB-Compatible HTTP API Layer over inGitDB

Implement a CouchDB-compatible HTTP server that fronts an inGitDB repository. The server
speaks the CouchDB wire protocol so that PouchDB (and any CouchDB client) can use inGitDB
as a sync target without modification.

**Flow:**

```
PouchDB (browser)  ←—CouchDB protocol—→  inGitDB CouchDB adapter  ←—→  inGitDB repo (Git)
```

**What inGitDB maps to:**

| CouchDB concept | inGitDB equivalent |
|---|---|
| Database | inGitDB collection |
| Document `_id` | Record key (filename without extension) |
| Document `_rev` | Git commit SHA or content hash |
| `_changes` feed | `git log` of the collection data directory |
| `_bulk_docs` | Batch write → single commit |
| `_design` views | inGitDB materialized views (partial mapping) |

**Pros:**
- Zero changes to existing PouchDB applications — they just point to a different sync URL.
- Unlocks the entire PouchDB ecosystem (offline-first apps, Fauxton UI, replication tools).
- inGitDB becomes a persistent, Git-backed, version-controlled sync target.
- Each PouchDB sync becomes a reviewable Git commit (data changes visible in PRs).
- Enables AI agents (via inGitDB MCP server) to read data that was written by browser apps.

**Cons:**
- CouchDB `_rev` semantics (MVCC, conflict documents) are complex to replicate faithfully
  on top of Git. A compatibility subset is more realistic than full CouchDB parity.
- CouchDB design documents (MapReduce views) map poorly to inGitDB materialized views.
- The CouchDB replication protocol has edge cases (attachment support, filtered replication,
  selector-based replication) that would require incremental implementation.

---

### Approach B — inGitDB ↔ CouchDB Sync Bridge

Implement a standalone sync bridge (a CLI command or daemon) that replicates data between
an inGitDB collection and a real CouchDB database. The bridge reads the inGitDB `_changes`
feed (via `git log`) and pushes to CouchDB, and vice versa.

**Flow:**

```
inGitDB repo  ←—sync bridge—→  CouchDB / PouchDB server
```

**Pros:**
- Simpler to implement — uses the existing CouchDB HTTP API as a client, not a server.
- No protocol emulation required.
- Useful for teams that already run CouchDB and want to mirror data into Git for auditability,
  code review, or AI access.

**Cons:**
- Requires a running CouchDB instance — adds infrastructure dependency.
- Does not unlock the PouchDB offline-first ecosystem (PouchDB still syncs to CouchDB,
  not to inGitDB directly).
- Two separate stores with a bridge are harder to keep consistent than a single source of truth.
- Less transformative — the value-add over a generic ETL tool is limited.

---

## Recommendation

**Implement Approach A** — a CouchDB-compatible HTTP API layer over inGitDB.

Rationale:

1. **Higher leverage.** The CouchDB replication protocol is well-documented and
   implemented in PouchDB. Building a compatible server makes inGitDB usable by every
   PouchDB application without code changes on the client side.

2. **Git as the source of truth.** With Approach A, the inGitDB Git repository is the
   canonical store. All writes go through Git, preserving history, enabling PR-based review,
   and making data available to AI agents via the MCP server. With Approach B, CouchDB and
   inGitDB are peers — eventual consistency complexity is inherent.

3. **Offline-first + Git history.** PouchDB is designed for offline-first apps. Pairing
   it with inGitDB as the sync server gives those apps Git-native versioning for free:
   every sync is a commit, every branch is a data snapshot.

4. **Incremental scope.** A minimal viable implementation requires only a subset of the
   CouchDB protocol (see phases below). Full CouchDB parity is not required to unlock
   practical value.

---

## Technical Design

### New Component: `inGitDB CouchDB Adapter`

A new inGitDB component (likely a sub-command of the CLI or a standalone binary) that:

1. Speaks HTTP on a configurable port.
2. Implements the CouchDB replication protocol endpoints.
3. Translates CouchDB operations to inGitDB Git operations.
4. Translates inGitDB Git history to a CouchDB-style changes feed.

### API Surface (MVP)

The minimal set of endpoints needed for PouchDB sync to work:

```
GET  /                          # Server info (welcome message + version)
GET  /{db}                      # Database info (doc_count, update_seq)
PUT  /{db}                      # Create database (ensure collection exists)
GET  /{db}/_all_docs            # List all documents (maps to collection records)
POST /{db}/_all_docs            # Fetch docs by keys
GET  /{db}/_changes             # Changes feed (maps to git log)
POST /{db}/_bulk_docs           # Batch write (maps to batch commit)
GET  /{db}/{docId}              # Get document by ID
PUT  /{db}/{docId}              # Create/update document
DELETE /{db}/{docId}            # Delete document
POST /{db}/_revs_diff           # Replication: diff revisions
POST /{db}/_bulk_get            # Replication: fetch docs by rev
GET  /{db}/_local/{docId}       # Replication checkpoint (store as hidden file)
PUT  /{db}/_local/{docId}       # Replication checkpoint
```

### `_rev` Strategy

CouchDB `_rev` encodes a generation counter (`1-`, `2-`, …) plus a content hash. inGitDB
can derive `_rev` from the Git commit SHA that last modified the record:

```
_rev = "<generation>-<sha1-of-content>"
```

- Generation is computed by counting commits that touch the record file (`git log --follow -- path`).
- Content hash is the SHA-1 of the document JSON.
- On conflict (two clients write the same document concurrently), follow CouchDB's
  deterministic conflict winner rule (highest rev string wins), and store losing revisions
  as Git branches or notes.

### Changes Feed Strategy

```
GET /{db}/_changes?since=<seq>
```

- `seq` maps to a Git commit SHA (or `0` for the beginning of history).
- The feed is constructed by `git log --reverse --since=<sha>` on the collection data directory.
- Each changed file in each commit becomes a change entry.
- `longpoll` and `continuous` feed modes require a running watcher (reuse the existing
  inGitDB Watcher component).

### Document → Record Mapping

| CouchDB document field | inGitDB record |
|---|---|
| `_id` | Record key (file basename) |
| `_rev` | Derived from git history (not stored in file) |
| `_deleted: true` | Record file deleted from Git |
| All other fields | Record data columns |

The `_rev` field is **not stored** in the inGitDB record file. It is computed on read and
validated on write, keeping record files clean and human-readable.

### Schema Mapping

CouchDB is schema-free; inGitDB is schema-validated. Two modes:

- **Strict mode:** The target collection must have a `.definition.yaml`. Documents that
  fail schema validation are rejected with a `400` error.
- **Flexible mode:** The target collection uses `type: any` for all fields, accepting any
  JSON structure. Schema can be added and tightened later.

---

## Implementation Phases

### Phase 1 — Read-Only Compatibility (PouchDB can pull from inGitDB)

Implement read endpoints:

- `GET /`, `GET /{db}`, `GET /{db}/_all_docs`, `GET /{db}/{docId}`
- `GET /{db}/_changes` (normal feed, no longpoll)
- `POST /{db}/_revs_diff`, `POST /{db}/_bulk_get`
- `GET/PUT /{db}/_local/{docId}` (replication checkpoints)

**Outcome:** PouchDB can replicate inGitDB data to the browser. Existing inGitDB repos
become a data source for offline-first apps.

### Phase 2 — Write Compatibility (PouchDB can push to inGitDB)

Add write endpoints:

- `PUT /{db}/{docId}`, `DELETE /{db}/{docId}`, `POST /{db}/_bulk_docs`

Map writes to Git commits with the message `couch: sync from <source>`.

**Outcome:** PouchDB can sync bidirectionally with inGitDB. Browser edits become Git commits.

### Phase 3 — Continuous Changes Feed

- `GET /{db}/_changes?feed=longpoll` and `?feed=continuous` (SSE / chunked HTTP)
- Powered by the existing inGitDB Watcher component.

**Outcome:** Live sync — browser sees inGitDB changes in real time.

### Phase 4 — Conflict Resolution

- Expose CouchDB-style conflict documents (`_conflicts` array).
- Integrate with the inGitDB Merge Conflict Resolver TUI.

**Outcome:** Full offline-first conflict semantics, resolved through Git tooling.

---

## Open Questions

1. **Where does this live?** Options:
   - Sub-command of `ingitdb-cli` (`ingitdb serve --couch`), extending the planned HTTP API server.
   - Standalone adapter package in `ingitdb-ts` (TypeScript, deployable as a Node.js server).
   - Separate `ingitdb-couch` repository.

2. **Authentication.** CouchDB uses HTTP Basic Auth and cookie-based sessions. inGitDB
   currently has no auth layer. Phase 1 can be unauthenticated (localhost only); a token
   or Basic Auth shim can be added in a later phase.

3. **Multi-collection routing.** CouchDB has a flat database namespace. inGitDB has a
   hierarchical collection namespace. The adapter could expose one CouchDB "database" per
   inGitDB collection, using URL-safe encoded paths (e.g., `group__collection`).

4. **PouchDB database adapter.** An alternative to running an HTTP server is to publish
   a custom PouchDB adapter (`pouchdb-adapter-ingitdb`) that reads/writes inGitDB directly
   via `@ingitdb/client-github` or `@ingitdb/client-fs`. This eliminates the HTTP layer for
   browser use cases. It could complement (not replace) the HTTP server approach.

5. **Design documents.** CouchDB `_design` documents with MapReduce views are widely used.
   A minimal compatibility shim could ignore unknown design docs (return `200 OK` with
   empty results) so replication does not break when design docs are replicated.

---

## References

- [CouchDB HTTP API Reference](https://docs.couchdb.org/en/stable/api/index.html)
- [CouchDB Replication Protocol](https://docs.couchdb.org/en/stable/replication/protocol.html)
- [PouchDB Documentation](https://pouchdb.com/guides/)
- [PouchDB Custom Adapters](https://pouchdb.com/guides/adapters.html)
- [inGitDB HTTP API Server feature spec](../docs/features/http-api-server.md)
- [inGitDB TypeScript Clients](../docs/components/clients.md)
