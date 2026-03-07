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
        │   │   └── <name>.yaml
        │   └── views/                       # View definitions
        │       └── <name>.yaml
        └── $views/
            └── <view-name>/
                └── <partition>.md         # Materialized view output files
```

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

## Collection Schema (.definition.yaml)

**`.definition.yaml`** — Collection schema: titles (i18n), column definitions, record file format.

```yaml
titles:
  en: Tasks
data_dir: $records
record_file:
  name: "$records/{key}.json"
  type: "[]map[string]any" # or "map[string]any" (single record) or "map[string]map[string]any" (keyed dict)
  format: json # or yaml
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

`string`, `int`, `float`, `bool`, `date`, `time`, `datetime`, `map[locale]string`, `any`

## Record Files

**Record files** live in the collection's `data_dir`. A file holds either one record (`map[string]any`) or an array of records (`[]map[string]any`), as declared in `record_file.type`.

## View Definitions

**View definitions** (`.collection/views/<name>.yaml`) declare how to partition and render records into materialized view files under `$views/`.

See [docs/schema/](schema/) for the full schema reference.
