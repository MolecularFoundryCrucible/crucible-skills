# crucible-skills

Claude Code Agent Skills for Crucible Data Platform workflows.

| Skill | What it covers |
|---|---|
| `add-or-update-dataset` | Creating and updating datasets — uploading files, scientific metadata, linking to parents and samples |
| `add-or-update-sample` | Registering and updating samples — derivation and containment links, provenance for derived metrics |
| `query-data` | Finding and counting records — field filters, fuzzy search, metadata search, relationships, aggregation |

Each skill is a folder containing a `SKILL.md`. Claude reads only the `name` and
`description` from the frontmatter until a skill triggers.

## Installing

Claude Code loads skills from two locations:

- `~/.claude/skills/` — available in every project
- `<project>/.claude/skills/` — available only in that project

You can symlink the skill files to the .claude/skills directory, so the repo stays the
single source of truth and edits go live when the repo is updated.

From the repo root:

```bash
mkdir -p ~/.claude/skills
for s in add-or-update-dataset add-or-update-sample query-data; do
  ln -sfn "$PWD/$s" ~/.claude/skills/"$s"
done
```

`ln -sfn` is safe to re-run — it replaces an existing symlink of the same name rather
than nesting inside it.

## Verifying

```bash
ls -l ~/.claude/skills/     # symlinks should resolve, not show as broken
```

Then start a **new** Claude Code session — skills are discovered at startup, so an
already-running session will not see them.

## Updating

Edit the `SKILL.md` in this repo. Changes take effect in the next Claude Code session —
no reinstall, because the symlinks point back here.

## Uninstalling

```bash
rm ~/.claude/skills/add-or-update-dataset \
   ~/.claude/skills/add-or-update-sample \
   ~/.claude/skills/query-data
```

Removes the symlinks only; the repo is untouched.
