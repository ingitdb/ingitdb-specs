# ⚙️ Subcollection Definition File

A **subcollection** is a collection nested within another collection's records. Subcollections use the exact same definition format as standard root-level collections (mapping to the [`CollectionDef`](https://github.com/ingitdb/ingitdb-go/blob/main/ingitdb/collection_def.go) type), with their placement defining their relationship to parent data.

## 📂 File location

The location of a subcollection's `definition.yaml` depends on which
[layout style](collection.md#-collection-layout-styles) the parent collection uses.

**Dedicated-directory layout (`.collection/`)**

Subcollections live under `.collection/subcollections/` inside the parent collection directory.
Each subcollection has its own named subdirectory containing a `definition.yaml`.

```
<parent-collection-dir>/
  .collection/
    definition.yaml
    subcollections/
      <sub>/
        definition.yaml
```

Nested subcollections mirror the same pattern one level deeper:

```
<parent-collection-dir>/
  .collection/
    subcollections/
      <sub>/
        definition.yaml
        subcollections/       ← sub's own subcollections
          <nested-sub>/
            definition.yaml
```

**Shared-directory layout (`.collections/`)**

Subcollections are **direct non-`$`-prefixed subdirectories** of the parent collection's entry
inside `.collections/`.

```
<base-dir>/
  .collections/
    <parent-name>/
      definition.yaml
      <sub>/                  ← subcollection (no "subcollections/" intermediate dir)
        definition.yaml
        <nested-sub>/         ← nested subcollection
          definition.yaml
```

The subcollection directory name is its ID. Any directory not prefixed with `$` is treated as
a subcollection. `data_dir` is resolved relative to the base directory (parent of
`.collections/`).

The subcollection directory name dictates the identity of the subcollection.

## 📂 Example

Using a company / organisation model is universally understood, cleanly hierarchical, and supports multiple subcollections at the same level without overlap.

### Data Structure

This model works well because of its clear containment rules. For example, `departments` and `offices` are multiple independent subcollections at the same level (under a company). `teams` and `projects` are nested subcollections inside `departments`, and `members` is further nested inside `teams`.

A minimal data sample for the structure above:

```text
companies
  acme-inc
    departments
      engineering
        teams
          backend
            members
              alice
              bob
        projects
          api-v2
      marketing
    offices
      dublin
      london
```

### Schema Structure

**Dedicated-directory layout** — schema definition files under the root `.collection/` folder:

```text
companies/
  .collection/
    definition.yaml                                   <-- "companies" schema
    subcollections/
      departments/
        definition.yaml                               <-- "departments" schema
        subcollections/
          projects/
            definition.yaml                           <-- "projects" schema (subset of departments)
          teams/
            definition.yaml                           <-- "teams" schema
            subcollections/
              members/
                definition.yaml                       <-- "members" schema
      offices/
        definition.yaml                               <-- "offices" schema
```

**Shared-directory layout** — the same hierarchy expressed under `.collections/`:

```text
companies/
  .collections/
    companies/
      definition.yaml                                 <-- "companies" schema
      departments/
        definition.yaml                               <-- "departments" schema
        projects/
          definition.yaml                             <-- "projects" schema
        teams/
          definition.yaml                             <-- "teams" schema
          members/
            definition.yaml                           <-- "members" schema
      offices/
        definition.yaml                               <-- "offices" schema
```

If you manage a `companies` collection and you want to track `departments` (a subcollection)
for each company, the `departments` definition would look exactly like a standard root-level
collection. The comment below shows the dedicated-layout path; adjust to
`.collections/companies/departments/definition.yaml` for the shared layout.

```yaml
# companies/.collection/subcollections/departments/definition.yaml
# (shared layout: companies/.collections/companies/departments/definition.yaml)
titles:
  en: Departments
record_file:
  name: "{key}/{key}.json"
  type: "map[string]any"
  format: json
columns:
  title:
    type: string
    required: true
  manager_id:
    type: string
```

## 📂 Where subcollection records live

A subcollection has one instance per parent record. The definition decides where it lives: in a
directory named by the parent key, **inside the parent's records directory**, next to the parent's
record file:

```
<parent data dir>/<parent records_dir>/<parentKey>/<sub>/
```

`<parent records_dir>` is the parent's `record_file.records_dir`: `.` means the data directory
itself, and when it is omitted it is `$records` for a `name` that contains `{key}` and the data
directory otherwise. Inside that directory the subcollection's own `record_file` applies, exactly
as for a root collection, and the rule repeats at every depth. This is the storage convention of
the ingitdb-go Feature
[subcollection-record-validation](https://github.com/ingitdb/ingitdb-go/blob/main/spec/features/subcollection-record-validation/README.md)
(REQ `subcollection-storage-convention`), with `records_dir` from
[explicit-record-base-directory](https://github.com/ingitdb/ingitdb-go/blob/main/spec/features/explicit-record-base-directory/README.md);
ingitdb-go is the source of truth for the schema. Every reader and writer follows it:
`ingitdb validate`, `ingitdb select` (including `--group`), and the storage drivers.

| Parent `records_dir` | Parent record | Its `contacts` instance |
| -------------------- | ------------- | ----------------------- |
| omitted, `name: "{key}.yaml"` (default `$records`) | `spaces/$records/family.yaml` | `spaces/$records/family/contacts/` |
| `.` | `spaces/family.yaml` | `spaces/family/contacts/` |
| `data` | `spaces/data/family.yaml` | `spaces/data/family/contacts/` |
| omitted, `name: "spaces.yaml"` (map of records) | key `family` in `spaces/spaces.yaml` | `spaces/family/contacts/` |

With `records_dir: .` this is the Firestore path, one to one:

```
spaces/k3v9q.json                                   <- record spaces/k3v9q
spaces/k3v9q/ext/calendarius.json                   <- record of subcollection spaces/ext
spaces/k3v9q/ext/calendarius/days/2026-11-24.json   <- record of subcollection spaces/ext/days
ext/commitius/cmSessions/s1.json                    <- ext/commitius.json does not exist
```

**The parent record file is optional.** As in Firestore, a parent may own subcollections without
existing itself: in the last line there is no `ext/commitius.json`. Validation (ingitdb-go) finds the
parent from the directory `ext/commitius/cmSessions/` and validates `s1.json` all the same, at any
depth. A missing parent is not counted as a record of the parent collection. No placeholder file is
needed (an empty `{}` parent file still works). Only a directory that contains the subcollection's
own directory counts as a parent. Hidden directories, the names inGitDB writes itself (`$records`,
`$views`, `$ingitdb`, the parent's own `records_dir`) and names matched by the parent's
`exclude_regex` never do; any other name does, including a Firestore-style id such as `$x`. A
parent directory that is a symbolic link counts only when it resolves inside the database; one that
leads outside is not read, and `ingitdb validate` reports it as a warning.

**Data in any other place is an error.** Earlier versions of the storage drivers wrote an instance
at `<parent data dir>/<parentKey>/<sub>/` whatever the parent's `records_dir`. When the parent's
`records_dir` is not `.`, files found there are not where the definition puts them:
`ingitdb validate` fails with one error per such directory, naming the collection, the found path,
the file count, the expected path and the fix, and `ingitdb select` and the drivers fail with the
same details instead of reading the instance as empty. For example, with the default `records_dir`:

```
spaces/family/contacts/c1.yaml          <- error: belongs at spaces/$records/family/contacts/c1.yaml
```

Fix it by moving the files to the expected path, or, if the collection should use the
Firestore-mirror layout, by setting `record_file.records_dir: "."` on the parent and moving its own
records out of `$records/`. A database that kept data in the old place passed validation before
(those records were never read) and fails now; that is intended.

`ingitdb validate` prints one count line per subcollection, by its full schema path (for example
`All 2 records are valid for collection: spaces/ext/days`), adding up all parent instances.

## 📂 Schema Format

The schema format for a subcollection is identical to a standard collection. Please refer to the [Collection Definition File](collection.md) for a comprehensive list of all supported fields, column types, and storage definitions.
