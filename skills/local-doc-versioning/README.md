# Local Doc Versioning

Git-like versioning for AI-generated feature docs that stays **100% local** and **doesn't pollute Git history**.

> For the full specification, see [SKILL.md](SKILL.md).

## What this skill is for

Use `local-doc-versioning` when you have AI-generated docs that change fast and you want:

- **snapshots** of doc state at checkpoints
- **recall** of an older state (to replay context / decisions)
- **diffs** between two states
- **plans** that map doc changes to code changes
- **apply** incremental code patches from doc diffs
- **retention + pinning** so storage doesn't grow forever

Non-goals:

- Long-lived specs that must be code-reviewed and checked into Git (put those in `docs/` and commit them).

## Installation

### Via Skills CLI

```bash
npx skills add jyunhanlin/ai-toolkit -s local-doc-versioning
```

### Manual

```bash
git clone https://github.com/jyunhanlin/ai-toolkit.git
cp -r ai-toolkit/skills/local-doc-versioning ~/.claude/skills/
```

Then restart Claude Code (or run `/refresh-skills`).

## Quick start

1. Put feature docs under `source_root` (default: `.claude/feature-docs/`).
2. Save snapshots at meaningful checkpoints:
   ```
   /local-doc-versioning save my-feature --message "Initial spec"
   ```
3. Use `list`, `recall`, `diff` to navigate history.
4. Use `plan` and `apply` to derive code changes from doc diffs.
5. Use `gc` and `pin` to manage storage.

First run auto-initializes `.agentdocs/`, config, and `.gitignore`.

## Commands

| Command | Description |
|---------|-------------|
| `save [feature_key] [--message "..."]` | Create a snapshot of all docs |
| `list [--limit N]` | Show recent snapshots |
| `recall <snapshot_id>` | Load a previous snapshot |
| `diff <from_id> <to_id>` | Compare two snapshots |
| `plan <from_id> <to_id>` | Derive code change plan from doc diff *(experimental)* |
| `apply <from_id> <to_id> [--dry-run]` | Apply code changes based on doc diff *(experimental)* |
| `gc [--keep-last N] [--keep-days D] [--dry-run]` | Clean up old snapshots |
| `pin <snapshot_id>` | Mark snapshot as permanent |
| `unpin <snapshot_id>` | Remove pin from snapshot |

Snapshot IDs look like `YYYYMMDD-HHMMSS-<hash>` (example: `20260205-170300-a3f2c1`).

## Configuration

Config lives at `.agentdocs/config.json`. Template defaults are in `template/config.json`.

```json
{
  "source_root": ".claude/feature-docs",
  "store_root": ".agentdocs",
  "retention": {
    "keep_last": 10,
    "keep_days": 7,
    "pins": []
  },
  "doc_format": {
    "require_stable_ids": true,
    "id_pattern": "REQ-\\d+"
  }
}
```

- **`source_root`**: where docs are read from
- **`store_root`**: where snapshots/metadata are stored (should be gitignored)
- **`retention.keep_last`** / **`retention.keep_days`**: keep the union of these rules
- **`retention.pins`**: snapshot IDs never deleted by GC
- **`doc_format.require_stable_ids`**: recommended for higher-quality diffs/recall
- **`doc_format.id_pattern`**: regex for stable IDs (default `REQ-\d+`)

## Storage layout

```
<project>/
  .claude/feature-docs/           # source docs
  .agentdocs/                     # local-only storage (gitignored)
    config.json
    snapshots/
      <snapshot_id>/
        manifest.json
        files/
          ...
    applied/                      # operation logs from apply command
```

## Guidelines

- **Use stable IDs in headings** so changes are trackable:
  ```markdown
  ## REQ-001: OAuth Implementation
  ## REQ-002: User Profile API
  ```
- **Save at checkpoints**: after a big generation, before a big rewrite, end of session.
- **Pin milestones**: feature done, demo, release notes.
- **Always review plans before apply**: run `plan` first, inspect, then `apply`.

## Troubleshooting

- **Snapshots not created**: ensure `source_root` exists and contains files.
- **Apply fails with conflicts**: conflict regions are marked with `<<<<<<<` markers — resolve manually.
- **Storage too large**: run `gc --keep-last 5 --keep-days 3`.
- **Snapshot not found**: run `list` to see available IDs. IDs are case-sensitive.
