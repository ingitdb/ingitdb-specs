# 📘 inGitDB Diff

`ingitdb diff` shows what changed between two points in a database's git history — which records were added, updated, or deleted — at a collection-level summary or full field-by-field detail.

## 📂 Status

Pending.

## 📂 What it does

Compares two git reference points (commits, branches, tags) and reports inGitDB record-level changes between them. Unlike `git diff`, which operates on raw file bytes, `ingitdb diff` understands the database structure: it groups file changes into record changes, maps fields, and annotates per-record output with the commit count and hashes that touched each record.

Results can be scoped to a specific collection, a path prefix, or a named view. For views, you can choose to inspect either the generated output file or the source records that feed into it.

## 📂 Reference points

Modeled after `git diff`. The positional argument accepts:

| Form | Meaning |
|------|---------|
| _(omitted)_ | `HEAD` vs. working tree (uncommitted changes) |
| `<ref>` | `<ref>` vs. `HEAD` — **primary use case**: compare a target branch against your current HEAD |
| `<ref>..<ref>` | Two explicit commits, branches, or tags |
| `<commit>` | `<commit>` vs. working tree |

The single-ref form (`ingitdb diff main`) is the primary use case: see what your current HEAD has changed relative to another branch.

**Examples:**

```
ingitdb diff main
ingitdb diff origin/main
ingitdb diff main..feature/add-regions
ingitdb diff v1.0..v2.0
ingitdb diff HEAD~5
```

## 📂 Scope filtering

By default all collections in the database are included. Two flags narrow the scope:

| Flag | Description |
|------|-------------|
| `--collection=KEY` | Limit to one collection (dot-notation, e.g. `countries.cities`). Mutually exclusive with `--view`. |
| `--path-filter=PATTERN` | Limit to records whose path matches the given prefix or glob (e.g. `countries/ie/*`). |

Both flags can be combined.

## 📂 Output depth

Controlled by `--depth`:

| Value | What you see |
|-------|-------------|
| `summary` _(default)_ | One line per **collection or view** with added/updated/deleted counts. |
| `record` | One line per **record** with change type, commit count, and short commit hashes. |
| `fields` | Per record + list of **field names** that changed (no values). |
| `full` | Per record + **before and after values** for every changed field. |

### `summary` depth (default)

```
Comparing main..HEAD  (4 commits)

  Collection          Added  Updated  Deleted
  ─────────────────── ─────  ───────  ───────
  countries               0        1        0
  countries.cities        2        1        1
```

### `record` depth

```
Comparing main..HEAD  (4 commits)

  countries
    ~ countries/ie                      [1 commit:  a1b2c3d]

  countries.cities
    + countries/ie/cities/cork          [1 commit:  a1b2c3d]
    + countries/fr/cities/nice          [1 commit:  a1b2c3d]
    ~ countries/gb/cities/london        [2 commits: e4f5a6b, 7c8d9e0]
    - countries/us/cities/old-town      [1 commit:  7c8d9e0]
```

Commit hashes are 7-character short SHAs.

### `fields` depth

```
  countries.cities
    ~ countries/gb/cities/london        [2 commits: e4f5a6b, 7c8d9e0]
        population
        area_km2
```

### `full` depth

```
  countries.cities
    ~ countries/gb/cities/london        [2 commits: e4f5a6b, 7c8d9e0]
        population:  892_000  →  900_000
        area_km2:    1_572    →  1_583
```

## 📂 View diffing

When scoping to a named view with `--view=VIEW_KEY`, the `--view-mode` flag selects what to compare:

| `--view-mode` value | Description |
|---------------------|-------------|
| `output` _(default)_ | Diff the generated view output file(s) between the two refs. |
| `source` | Diff the source records that feed into the view between the two refs. |

Run the command twice with different `--view-mode` values to see both the effect (output) and the cause (source records). The `--depth` flag applies to both modes.

## 📂 Output formats

| `--format` value | Description |
|------------------|-------------|
| `text` _(default)_ | Human-readable terminal output. |
| `json` | Structured JSON for scripting and tooling. |
| `yaml` | YAML output. |
| `toml` | TOML output. |

### JSON structure

At `summary` depth:

```json
{
  "from": "main",
  "to": "HEAD",
  "total_commits": 4,
  "collections": {
    "countries":        { "added": 0, "updated": 1, "deleted": 0 },
    "countries.cities": { "added": 2, "updated": 1, "deleted": 1 }
  }
}
```

At `record` depth each collection entry gains `"records"` arrays:

```json
{
  "from": "main",
  "to": "HEAD",
  "total_commits": 4,
  "collections": {
    "countries.cities": {
      "added": 2, "updated": 1, "deleted": 1,
      "records": {
        "added": [
          { "path": "countries/ie/cities/cork",     "commits": ["a1b2c3d"] },
          { "path": "countries/fr/cities/nice",     "commits": ["a1b2c3d"] }
        ],
        "updated": [
          { "path": "countries/gb/cities/london",   "commits": ["e4f5a6b", "7c8d9e0"] }
        ],
        "deleted": [
          { "path": "countries/us/cities/old-town", "commits": ["7c8d9e0"] }
        ]
      }
    }
  }
}
```

At `fields` depth each record entry gains a `"fields"` string array. At `full` depth `"fields"` becomes an array of objects:

```json
{
  "from": "main",
  "to": "HEAD",
  "total_commits": 4,
  "collections": {
    "countries.cities": {
      "added": 2, "updated": 1, "deleted": 1,
      "records": {
        "added": [
          {
            "path": "countries/ie/cities/cork",
            "commits": ["a1b2c3d"],
            "fields": [
              { "name": "name",       "from": null, "to": "Cork"   },
              { "name": "population", "from": null, "to": 210000   }
            ]
          }
        ],
        "updated": [
          {
            "path": "countries/gb/cities/london",
            "commits": ["e4f5a6b", "7c8d9e0"],
            "fields": [
              { "name": "population", "from": 892000, "to": 900000 },
              { "name": "area_km2",   "from": 1572,   "to": 1583   }
            ]
          }
        ],
        "deleted": [
          {
            "path": "countries/us/cities/old-town",
            "commits": ["7c8d9e0"],
            "fields": [
              { "name": "name",       "from": "Old Town", "to": null },
              { "name": "population", "from": 45000,      "to": null }
            ]
          }
        ]
      }
    }
  }
}
```

## 📂 Exit codes

| Code | Meaning |
|------|---------|
| `0` | Diff completed; no changes found. |
| `1` | Diff completed; one or more changes found. |
| `2` | Infrastructure error (git not found, invalid ref, bad flags). |

Exit code `1` on any change found makes `ingitdb diff` usable directly in CI guard scripts.

## 📂 HTTP API

`ingitdb diff` is exposed as an HTTP endpoint by the inGitDB API server:

```
GET /repos/{owner}/{repo}/diff
```

Query parameters mirror the CLI flags:

| Parameter | CLI equivalent | Description |
|-----------|---------------|-------------|
| `from` | first ref in `<ref>..<ref>` | From-ref (branch, tag, commit SHA). Defaults to the repository's initial commit. |
| `to` | second ref / `HEAD` | To-ref. Defaults to `HEAD`. |
| `collection` | `--collection=KEY` | Limit to one collection (dot-notation). |
| `path_filter` | `--path-filter=PATTERN` | Limit to matching record paths. |
| `view` | `--view=VIEW_KEY` | Scope to a named view. |
| `view_mode` | `--view-mode=output\|source` | View diff mode. Default: `output`. |
| `depth` | `--depth=summary\|record\|fields\|full` | Output depth. Default: `summary`. |

Response is always JSON in the same structure as `ingitdb diff --format=json`. HTTP status is `200` whether changes are found or not; the presence of changes is determined by the response body.

**Example:**

```
GET /repos/acme/products-db/diff?from=main&to=HEAD&collection=products&depth=record
```

## 📂 MCP Server

`ingitdb diff` is exposed as an MCP tool named **`ingitdb_diff`**:

```json
{
  "name": "ingitdb_diff",
  "description": "Show record-level changes between two git refs in an inGitDB repository.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "owner":       { "type": "string", "description": "Repository owner." },
      "repo":        { "type": "string", "description": "Repository name." },
      "from":        { "type": "string", "description": "From-ref (branch, tag, or commit SHA)." },
      "to":          { "type": "string", "description": "To-ref. Defaults to HEAD." },
      "collection":  { "type": "string", "description": "Limit to one collection (dot-notation)." },
      "path_filter": { "type": "string", "description": "Limit to matching record paths." },
      "view":        { "type": "string", "description": "Scope to a named view." },
      "view_mode":   { "type": "string", "enum": ["output", "source"], "description": "View diff mode." },
      "depth":       { "type": "string", "enum": ["summary", "record", "fields", "full"], "description": "Output depth." }
    },
    "required": ["owner", "repo"]
  }
}
```

The tool returns a JSON object in the same structure as `ingitdb diff --format=json`. AI agents can use it to answer questions like "what changed in the products collection since the last release?" or "show me every field that changed on record X".

## 📂 Related

- [Pull](pull.md) — uses the same record-change logic to print its post-pull change summary.
- [Watcher](watcher.md) — emits similar record-change events in real time for file-system changes.
- [Migration Script Generator](migration.md) — uses diff output to generate forward/rollback migration scripts.
- [HTTP API Server](http-api-server.md) — the server that hosts the `/diff` endpoint.
- [MCP Server](mcp-server.md) — the MCP server that hosts the `ingitdb_diff` tool.
