---
name: add-or-update-sample
description: Create or update a sample in the Crucible Data Platform — registering a
  physical specimen, recording its scientific metadata, and linking it to the datasets
  measured from it, to the samples it came from, and to the batch, tray, wafer, or grid
  it belongs to. Use whenever the user wants a specimen, substrate, film, device, or
  batch added, saved, created, or registered in Crucible, or wants to change the name,
  type, metadata, or relationships of a sample already in Crucible. This skill is for
  samples specifically — use the dataset skill when the user is describing acquired data
  or a synthesis recipe rather than a physical specimen.
license: MIT
---

# Add or update a Crucible sample

A sample is Crucible's record of a physical specimen: what it is, what it was made
from, what it belongs to, and what was measured from it.

Use the `crucible` CLI for every operation in this skill.

## Before you start

Run `crucible whoami` to confirm authentication and `crucible config show` to see the
default project. If either fails, point the user at `crucible config init` and stop.

If the user has not named a project and no `current_project` is configured, ask.

## Step 1 — Check whether the sample already exists

```bash
crucible sample list -pid <PROJECT> --limit 20
```

If a sample with the same name is already in the project, tell the user and ask whether
they want to update the existing record or create a second one.

## Step 2 — Assemble the sample fields

| Field | Flag | The user might say |
|---|---|---|
| `sample_name` | `-n` | name, label, what it's called |
| `project_id` | `-pid` | project, proposal |
| `sample_type` | `--type` | type, kind, what it is |
| `description` | `--description` | description, notes |
| keywords | `-k` | keywords, tags, labels (comma-separated) |

You may infer values from context, but confirm anything inferred with the user before
creating the sample.

Do not open data files to extract metadata. That is the ingestor's job.

For `sample_type`, retrieve the values already in use and reuse an existing spelling
where it matches — the field is free text, and inconsistent spellings fragment search
results.

```bash
crucible sample list -pid <PROJECT> --limit 20
```

## Step 3 — Identify the relationships

Samples relate to other samples through parent/child links. Two distinct cases, both
expressed the same way:

- **Derivation** — this sample was physically made from another. "Cut from wafer A",
  "annealed from S-102", "this is S-102 after the second deposition." The precursor is
  the parent.
- **Containment** — this sample is part of a batch, tray, container, bar, grid, or
  wafer. The container is the parent.

The user's wording is the signal: *"created from X and Y"* is derivation, *"part of
tray Z"* is containment. If it is not clear whether a relationship exists, ask.

For each named parent, check whether it has a Crucible record. If not, offer to create
it — do not create parents silently.

## Step 4 — Build the scientific metadata

Only build `scientific_metadata` if the user explicitly asks for it. Most metadata
belongs on the related dataset, not the sample.

If the user gives a metric that looks calculated or measured rather than recorded by
hand — an area, a yield, a thickness, a rate — it probably came from a dataset. Look at
what is already linked to the sample and ask the user whether the value derives from one
of them:

```bash
crucible sample list-datasets <SAMPLE_MFID>
```

If they confirm, store the dataset ID alongside the value so the provenance survives:

```bash
--metadata '{"outgassing_area": 4.5e-14, "derived_from_dataset": "<DSID>"}'
```

## Step 5 — Dry run, then create

```bash
crucible sample create -n "S-102" -pid my-project \
  --type "thin film" \
  -k "sputtered,batch-7" \
  --dry-run
```

Show the user what the dry run reports, then re-run without `--dry-run`.

## Step 6 — Link parent samples

```bash
crucible link -p <PARENT_SAMPLE_MFID> -c <NEW_SAMPLE_MFID>
```

Both derivation and containment use this command. Batch and container samples are
ordinary samples that happen to have many children.

## Step 7 — Link datasets

```bash
crucible link -d <DSID> -s <SAMPLE_MFID>
```

If the user described a synthesis recipe, a deposition run, or any other procedure that
produced this sample, that is a dataset — create it with the dataset skill and link it
here.

Finish with `crucible open <SAMPLE_MFID>` to show the user the result in the Graph
Explorer.

---

## Working with related entities

This skill and the dataset skill each describe the full picture, so following both
literally could loop: create a sample → notice a dataset is needed → create the dataset →
notice a sample is needed.

The rule is simple: **finish the thing the user asked for, then attach what it needs.**
Whatever you are creating right now is the subject. Related entities are looked up
first; only create one if it genuinely does not exist, and only after the user confirms.
Once something exists, it is a link target — never revisit it as a new subject.

---

## Updating an existing sample

**Fields** — repeatable `--set KEY=VALUE`:

```bash
crucible sample update <SAMPLE_MFID> --set sample_name="Corrected name"
```

**Scientific metadata** — merges into what is already there by default:

```bash
crucible sample update <SAMPLE_MFID> --metadata '{"notes": "recut"}'
```

`--overwrite` replaces the entire dictionary instead of merging. Before using it, make
sure the user understands that overwrite will completely replace the existing metadata
dictionary.
