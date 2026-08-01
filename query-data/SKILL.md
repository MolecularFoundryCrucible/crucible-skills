---
name: query-data
description: Find and count data in the Crucible Data Platform — listing datasets and
  samples by field, fuzzy-searching names, full-text searching scientific metadata,
  walking parent/child relationships, and assembling counts or summaries. Use whenever
  the user asks which, how many, show me, find, look up, list, or search for datasets,
  samples, files, instruments, users, or projects in Crucible, or asks about the
  experimental parameters, instrument settings, synthesis recipe or steps, or
  characterization results recorded for something in Crucible. This skill is read-only —
  use the dataset or sample skills when the user wants to create or change a record.
license: MIT
---

# Query Crucible data

Crucible offers several query surfaces that look interchangeable and are not. Picking
the wrong one usually returns a plausible, wrong answer rather than an error, so choose
deliberately.

Use the `crucible` CLI for simple lookups. Reach for the Python client whenever the
question needs a filter, a project scope, or an aggregation the CLI cannot express —
which is often, and is not a failure mode.

## Before you start

Run `crucible whoami` to confirm authentication. If it fails, point the user at
`crucible config init` and stop.

Every query is scoped to what your credentials can read. "No results" can mean the data
does not exist *or* that you cannot see it — do not report absence as fact without
saying which you checked.

## Pick the query surface

| The user wants | Use |
|---|---|
| A specific record, ID in hand | `crucible get <MFID>` |
| Records where a field equals a known value | `crucible dataset list` / `sample list`, or the Python client for fields the CLI does not expose |
| Records whose name they half-remember | `crucible dataset search` / `sample search` |
| Records where something appears in scientific metadata | `crucible dataset search-metadata`, then verify |
| Experimental parameters, settings, or results for a record | `crucible get <MFID> --include-metadata` |
| What a record is connected to | `crucible tree <MFID>` |
| A count, or a breakdown by field | Python client — see *Counting* |

## Resolving what the user named

**Projects.** When the user names a project, treat what they said as the `project_id`
and use it directly. Do not look up the project object first to validate it. Only if the
query returns an error or nothing does it become worth resolving:

```bash
crucible project search <TERM>    # matches project_id and title
```

**Users.** For "datasets Ron uploaded", search for the person, then filter by their ID:

```bash
crucible user search ron          # returns username, first/last name, orcid
```

```python
client.datasets.list(owner_orcid="<orcid from search>", limit=None)
```

`owner_orcid` holds a Crucible user ID, not a public ORCID number — use whatever the
search returned verbatim.

Do not use `crucible user list-datasets` for this. It lists datasets a user can
*access*, not datasets they own, and for most users that is nearly everything.

**When the project is unknown or the question spans projects.** `crucible dataset list`
and `crucible sample list` require a project — `-pid` or `current_project` — and will
refuse without one. The Python client has no such requirement, so use it to query across
everything the user can read:

```python
client.datasets.list(measurement="XRD", limit=None)          # all accessible projects
```

If the user does not know which project holds their data, or is asking a question that
spans projects, say so and query without a project filter rather than guessing one. To
scope to a set, list their projects and iterate:

```python
for p in client.projects.list():
    ...
```

## Getting a single record

```bash
crucible get <MFID>                    # type auto-detected
crucible get <MFID> --include-metadata # scientific metadata is omitted by default
crucible get <MFID> -v                 # all fields
```

Scientific metadata and links are **not** included unless asked for. Questions about
experimental parameters, instrument settings, synthesis steps, or results are answered
from scientific metadata — always pass `--include-metadata` for those, or you will
report that a record has none when it does.

## Filtering by field

```bash
crucible dataset list -pid <PROJECT> --instrument themis -m "STEM Imaging" --limit 50
crucible sample list -pid <PROJECT> --type wafer
```

The CLI exposes only a subset of the filters the API accepts:

| | CLI flags | Also filterable via the Python client |
|---|---|---|
| Dataset | `-pid/--project-id`, `-m/--measurement`, `-k/--keyword`, `--session`, `--data-format`, `--data-type`, `--instrument` | `unique_id`, `dataset_name`, `owner_orcid`, `source_folder`, `timestamp`, `size`, `public` |
| Sample | `-pid/--project-id`, `-n/--name`, `--type` | `unique_id`, `owner_orcid`, `description`, `timestamp`, `public` |

Any scalar column on the record can be passed as a filter kwarg to `list()` and
`count()`. Relationship names (`samples`, `keywords`, `parents`, `children`) are not
filters — do not pass them.

**All filters are exact and case-sensitive except `-k/--keyword`**, which is a
case-insensitive substring match. `--instrument Themis` finds nothing if the stored
value is `themis`. When a filtered query comes back empty, re-run without the filter and
check the real spelling before telling the user there is no such data.

**A misspelled filter name is silently ignored.** The server applies only the parameters
it recognises and drops the rest without complaint, so a typo returns the *unfiltered*
list, which looks like a successful query. This applies to CLI flags, `list()` kwargs,
and `count()` kwargs alike. Use the exact names above.

### Name globs and grouping happen after the fetch

`--include`, `--exclude`, and `--group-by` are applied by the CLI to the rows it already
fetched, *after* `--limit` has truncated them. `--include "run-*"` on the default limit
of 100 filters the newest 100 datasets in the project — not every dataset named `run-*`.
Raise `--limit` before trusting either.

Globs match the whole name, so use `*XRD*`, not `XRD`.

### Date ranges

The API supports `creation_time_gte`, `creation_time_lte`, `modification_time_gte`, and
`modification_time_lte`, but no CLI flag exposes them:

```python
client.datasets.list(project_id="my-project",
                     creation_time_gte="2026-03-01",
                     creation_time_lte="2026-04-01",
                     limit=None)
```

## Fuzzy name search

Trigram similarity — tolerates typos and word order, and will miss things a plain
substring match would catch. Minimum 3 characters, maximum 50 results.

**What each one actually searches differs by resource:**

| Command | Matches against |
|---|---|
| `crucible dataset search` | `dataset_name` only |
| `crucible sample search` | `sample_name` only |
| `crucible project search` | `project_id` and `title` |
| `crucible instrument search` | `instrument_name`, `instrument_type`, `manufacturer` |
| `crucible user search` | username, first name, last name |

Dataset and sample search do **not** look at measurement, instrument, project, or
metadata. To find datasets by instrument or measurement, filter with `dataset list`
instead — search will not find them.

For datasets and samples, if `current_project` is set in the config, search is
**silently scoped to that project** even though no `-pid` was given. Pass `-pid`
explicitly when the user means one project, and be aware you may be missing
cross-project hits when they did not.

## Scientific metadata search

```bash
crucible dataset search-metadata "thermal conductivity"
```

Full-text search over the metadata document. Its limits matter more than its
capabilities:

- Terms are stemmed and **ANDed**. No `OR`, no quoted phrases, no wildcards, no `NOT`.
- **No value comparison.** "Datasets above 300 K" is not expressible. `300` matches the
  token `300` and nothing else — not 301, not a range.
- **No key scoping, and key names are themselves indexed.** The whole metadata document
  is serialised to text before indexing, so searching `temperature` matches records that
  merely *have* a key named `temperature`, whatever its value. Never report a hit as
  meaning a value matched.
- **Not scoped by project**, and not scoped by resource type despite the command name —
  `dataset search-metadata` and `sample search-metadata` call the same endpoint and both
  return datasets, samples, instruments, and projects mixed together.
- Results carry only the ID and the metadata, not the name or type. Resolve each hit
  with `crucible get <MFID>` before describing it to the user.
- It is slow, and times out against large deployments. If it hangs, fall back to the
  fetch-and-filter approach below rather than retrying.

Treat it as a way to *find candidates*, then verify. When the user needs a precise
metadata condition, fetch with metadata included and filter in Python:

```python
rows = client.datasets.list(project_id="my-project", include_metadata=True, limit=None)
hot = [r for r in rows if (r.get("scientific_metadata") or {}).get("stage_temp_c", 0) > 300]
```

This is the only way to express a numeric comparison, a key-specific match, or an `OR`.

## Relationships

```bash
crucible tree <MFID>              # ancestors + full descendant tree
crucible tree <MFID> --all        # mix datasets and samples
crucible dataset list-samples <DSID>
crucible dataset list-parents <DSID>
crucible dataset list-children <DSID>
crucible sample list-datasets <SAMPLE_MFID>
crucible sample list-parents <SAMPLE_MFID>
crucible sample list-children <SAMPLE_MFID>
```

By default `crucible tree` shows only nodes of the **same type as the queried record**,
contracting paths that pass through the other type into direct edges. A dataset's linked
samples are therefore invisible in a default tree, and two datasets may appear directly
connected when in fact they are joined through a sample. Pass `--all` whenever the
relationship between datasets and samples is what the user is asking about.

## Counting and aggregating

`count` reads the server's total without fetching rows — far cheaper than listing:

```python
client.datasets.count(project_id="my-project", instrument_name="themis")
```

It takes the same filter kwargs as `list()`, **including the same silent-ignore
behaviour**. Sanity-check any count that looks suspiciously round or large.

There is no GROUP BY endpoint. `--group-by` on the CLI only groups the rows already
displayed. Real aggregation is assembled client-side — either a count per known value:

```python
{inst: client.datasets.count(project_id=pid, instrument_name=inst)
 for inst in ("themis", "team1")}
```

or one pass over the full list when the grouping values are not known up front:

```python
from collections import Counter
rows = client.datasets.list(project_id=pid, limit=None)
Counter(r.get("measurement") for r in rows)
```

`limit=None` fetches everything matching, following pagination automatically. Prefer it
over a large explicit limit when counting, and warn the user before doing it unscoped or
on a large project.

## Reporting results

Say what you actually ran. Given "how many XRD datasets do we have", the honest answer
names the scope — a count of `measurement="XRD"` in one project is not the same claim as
a count across every project the user can read, and they cannot tell which they got
unless you say so.

If a result set hit the limit, say it was truncated rather than presenting it as
complete.
