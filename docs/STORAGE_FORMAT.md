# inGitDB Storage Format

## Overview

inGitDB is a database system where a Git repository is the datastore. Records are YAML or JSON files; schema is declared in YAML config files placed alongside the data.

## Directory Layout

A database is a directory tree inside a Git repository:

```
<db-root>/
├── .ingitdb/                              # DB-level config
│   ├── root-collections.yaml             #   collection ID → path map
│   └── settings.yaml                     #   default_namespace, languages
└── <group>/
    └── <collection>/
        ├── .collection/
        │   ├── definition.yaml               # Collection schema
        │   ├── subcollections/              # Subcollection definitions
        │   │   └── <name>/definition.yaml
        │   └── views/                       # View definitions
        │       └── <name>.yaml
        ├── $records/                        # record files (default when name has {key})
        │   ├── <key>.json
        │   └── <key>/<sub>/...              # records of subcollection <sub> for parent <key>
        └── $views/
            └── <view-name>/
                └── <partition>.md         # Materialized view output files
```

The records of subcollection `<sub>` for parent `<key>` sit in `<records dir>/<key>/<sub>/`, next to
the parent's record file: `$records/<key>/<sub>/` by default, `<key>/<sub>/` with `records_dir: .`.
See [Where subcollection records live](schema/subcollection.md#-where-subcollection-records-live).

## Config Files

**`.ingitdb/root-collections.yaml`** — DB-level config: flat map of collection IDs to filesystem paths.

**`.ingitdb/settings.yaml`** — optional settings: `default_namespace` and supported languages.

```yaml
# .ingitdb/root-collections.yaml
companies: demo-dbs/test-db/companies
todo.*: demo-dbs/todo
agile.*: demo-dbs/agile-ledger
```

```yaml
# .ingitdb/settings.yaml
languages:
  - required: en
  - required: fr
  - optional: ru
```

## Collection Schema (`.collection/definition.yaml`)

**`.collection/definition.yaml`** — Collection schema: titles (i18n), column definitions, record file format.

```yaml
titles:
  en: Tasks
record_file:
  name: "{key}.json" # records go to $records/ unless records_dir says otherwise
  type: "map[string]any" # or "[]map[string]any" (list in one file) or "map[$record_id]map[$field_name]any" (keyed dict)
  format: json # or yaml, yml, markdown, toml, ingr, csv, jsonl
columns:
  title:
    type: string
    required: true
    min_length: 1
    max_length: 100
    titles:
      en: Task title
  status:
    type: string
    required: true
    foreign_key: statuses # value must be a valid record ID in the 'statuses' collection
```

## Column Types

`string`, `int`, `float`, `bool`, `date`, `time`, `datetime`, `any`, `map[locale]string`, `map[K]V`, and lists `[]T`. See [schema/collection.md](schema/collection.md#-column-types).

## Record Files

**Record files** live in the collection's `data_dir`, below `record_file.records_dir`. A file holds one record (`map[string]any`), a list of records (`[]map[string]any`), or a map of records keyed by id (`map[$record_id]map[$field_name]any`), as declared in `record_file.type`.

A subcollection's records live in `<parent records dir>/<parentKey>/<sub>/`. The parent record file is optional, as in Firestore. See [schema/subcollection.md](schema/subcollection.md#-where-subcollection-records-live).

## View Definitions

**View definitions** (`.collection/views/<name>.yaml`) declare how to partition and render records into materialized view files under `$views/`.

See [docs/schema/](schema/) for the full schema reference.
