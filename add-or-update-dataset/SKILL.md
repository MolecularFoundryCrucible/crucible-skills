---
name: add-or-update-dataset
description: Create or update a dataset in the Crucible Data Platform — uploading data
  files, recording scientific metadata, and linking the dataset to parent datasets and
  samples. Use whenever the user wants experimental data, measurement files, or an
  instrument run saved, uploaded, ingested, archived, or registered in Crucible, or
  wants to change the metadata, name, keywords, or files of a dataset already in
  Crucible. This skill is for datasets specifically — use the sample skill when the
  user is describing a physical specimen rather than acquired data.
license: MIT
---

# Add or update a Crucible dataset

A dataset is Crucible's unit of experimental record: the data files, the metadata
describing how they were produced, and links to related samples and datasets.

Use the `crucible` CLI for every operation in this skill.

## Before you start

Run `crucible whoami` to confirm authentication and `crucible config show` to see the
default project. If either fails, point the user at `crucible config init` and stop.

If the user has not named a project and no `current_project` is configured, ask.

## Step 1 — Inventory the files

Determine which files belong to this dataset. A dataset may contain one or more files,
or none at all.

If it is not clear which files should be grouped into one dataset and which should
become separate datasets, ask the user. Uploading twenty files as one dataset when they
meant twenty datasets is tedious to unwind.

## Step 2 — Check for an existing dataset identifier

An MFID is a 26-character lowercase Crockford Base32 string over the alphabet
`0123456789abcdefghjkmnpqrstvwxyz` (no `i`, `l`, `o`, or `u`). Example:
`0tkbkapkg1rgs0007h38dzayt0`.

Some acquisition systems record the dataset's intended MFID in the filename or in the
file's own metadata. If one is present, pass it with `--mfid <ID>` so the record matches
the identifier already on the sample label, logbook, or QR code.

**Matching the format is not enough.** Files often carry sample IDs, batch IDs, and
other MFIDs that are not this dataset's identifier. Only use a value that is clearly
identified as a run ID or dataset ID — a key named `run_id`, `dataset_id`, or an obvious
equivalent.

If you find a key that sounds like a run or dataset identifier but does not match those
exact names, ask the user whether it is the dataset MFID. When they answer, add the key
name to this skill so the next run recognises it without asking.

Never construct an MFID by hand. The alphabet excludes look-alike characters to survive
transcription, and a hand-made string will not be a valid encoded UUID.

## Step 3 — Check whether this dataset already exists

**By MFID.** If step 2 produced one, `crucible dataset get <MFID>` tells you whether the
record exists. Creating a dataset with an MFID that is already taken returns an error —
it does not create a duplicate.

**By file hash.** Ask the user whether to check; hashing large files takes time.

```bash
shasum -a 256 /path/to/file           # take the hex digest
crucible file list --sha256 <DIGEST>
```

Each returned file record carries a `dataset_mfid` identifying the dataset that already
holds that content.

If either check finds a match, ask the user how to proceed:

- **Skip** — already there.
- **Update** — keep the record, apply new metadata (see *Updating an existing dataset*).
- **Add files to it** — same dataset, more files.
- **Create anyway** — distinct data that happens to share a file.

## Step 4 — Determine the ingestor

Ingestors are server-side parsers that extract scientific metadata and generate
thumbnails after upload.

Set `ingestion_class` to `None` so the server auto-detects from file content. Specify an
ingestor only when the user has named a specific ingestion class they want. To show them
what is available, run `crucible dataset ingestors`.

## Step 5 — Assemble the dataset fields

Populate fields only from what the user actually provided. Do not infer values from
filenames, directory paths, or file contents at this step.

Users describe fields loosely, so interpret their wording:

| Field | Flag | The user might say |
|---|---|---|
| `dataset_name` | `-n` | name, title, what to call it |
| `project_id` | `-pid` | project, proposal |
| `instrument_name` | `--instrument` | instrument, microscope, tool, the machine |
| `measurement` | `-m` | measurement, technique — the industry-general term for what was measured |
| `data_type` | `--data-type` | data type — more specific than `measurement` |
| `session_name` | `--session` | session, sitting |
| `timestamp` | `--timestamp` | when it was taken (`today`, `2024-01-15`, ISO 8601) |
| keywords | `-k` | keywords, tags, labels (comma-separated) |

`public` is always false. Set `--public` only when the user explicitly asks to make the
data public.

For `instrument_name`, `measurement`, and `data_type`, retrieve the values already in
use and check whether the user's wording matches one of them. If it does, use the
existing value rather than a new spelling — these fields are free text, and inconsistent
spellings fragment search results.

```bash
crucible instrument list
crucible dataset list -pid <PROJECT> --limit 20
```

For instruments, the canonical spellings are `themis` and `team1`. Two stale variants
appear elsewhere in the codebase and should not be used for new datasets: `themisx` and
`team01`, both in `crucible-ingestion/src/constants.py`. Note `team05` is a different
instrument, not a variant of `team1`.

If a value the user gives does not match anything in use and you cannot tell whether it
is a new instrument or a misspelling of an existing one, ask.

## Step 6 — Build the scientific metadata

`scientific_metadata` is a free-form JSON dictionary.

Do not read the data files to populate it. The ingestor extracts instrument metadata
from the file after upload, and anything you add here by hand risks contradicting it.

Include anything that the user asked you to include in the metadata or scientific
metadata for the dataset.

```bash
--metadata '{"stage_temp_c": 300, "atmosphere": "vacuum", "notes": "drift after 20min"}'
--metadata ./run-metadata.json
```

## Step 7 — Dry run, then create

```bash
crucible dataset create -i sample.dm4 -pid my-project \
  -n "HAADF survey" --instrument themis \
  --metadata '{"stage_temp_c": 300}' \
  -k "haadf,survey" \
  --dry-run
```

Show the user what the dry run reports, then re-run without `--dry-run`.

`-i` accepts multiple paths and globs (`-i *.dm4`), producing **one** dataset containing
all of them. Loop if the user wanted one dataset per file.

## Step 8 — Confirm ingestion

Creation returns before server-side ingestion finishes; metadata and thumbnails appear
only after it completes.

```bash
crucible dataset ingestion <DSID>
```

If ingestion failed, report it and continue to the linking steps. The dataset exists and
its files are attached — it simply has no extracted metadata, and the links are still
worth making.

## Step 9 — Link parent datasets

Only if this data derives from another dataset.

```bash
crucible link -p <PARENT_DSID> -c <NEW_DSID>
```

## Step 10 — Link samples

Only if a related sample exists or the user names one.

```bash
crucible link -d <DSID> -s <SAMPLE_MFID>
```

Some ingestors create and link samples during ingestion, so check what is already linked
before adding more: `crucible dataset list-samples <DSID>`.

If the user names a sample with no Crucible record, offer to create it with
`crucible sample create` and link it.

Finish with `crucible open <DSID>` to show the user the result in the Graph Explorer.

---

## Updating an existing dataset

Fields and metadata update separately.

**Fields** — repeatable `--set KEY=VALUE`, auto-cast to int/float/bool/string:

```bash
crucible dataset update <DSID> --set dataset_name="Corrected name"
```

Editable: `dataset_name`, `measurement`, `data_type`, `session_name`, `instrument_name`,
`project_id`, `timestamp`, `description`, `public`.

Not editable: `unique_id`, `owner_orcid`, `data_format`, `size`, `creation_time`,
`modification_time`.

**Scientific metadata** — merges into what is already there by default:

```bash
crucible dataset update <DSID> --metadata '{"notes": "recalibrated"}'
```

`--overwrite` replaces the entire dictionary instead of merging, discarding
ingestor-extracted metadata. Before using it, make sure the user understands that
overwrite will completely replace the existing metadata dictionary.

**Adding files:**

```bash
crucible dataset add-file <DSID> -i newfile.dm4
```
