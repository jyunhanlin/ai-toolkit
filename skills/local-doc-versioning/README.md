# Local Doc Versioning

Git-like versioning for AI-generated feature docs that stays **100% local** and **doesn't pollute Git history**.

> For the full specification, see [SKILL.md](SKILL.md).

## What this skill is for

Use `local-doc-versioning` when you have AI-generated docs that change fast and you want:

- **snapshots** of doc state at checkpoints
- **recall** of an older state (to replay context / decisions)
- **diffs** between two states
- **plans** that map doc changes to code changes *(experimental)*
- **apply** incremental code patches from doc diffs *(experimental)*
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
4. Use `plan` and `apply` to derive code changes from doc diffs *(experimental)*.
5. Use `gc` and `pin` to manage storage.

First run auto-initializes `.agentdocs/`, config, and `.gitignore`.

## Commands

| Command | Description |
|---------|-------------|
| `save [feature_key] [--message "..."]` | Create a snapshot of all docs |
| `list [--feature <key>] [--limit N]` | Show recent snapshots |
| `recall <snapshot_id> [--content] [--restore]` | View (or restore) a previous snapshot |
| `diff <from_id> <to_id>` | Compare two snapshots |
| `plan <from_id> <to_id>` | Derive code change plan from doc diff *(experimental)* |
| `apply <from_id> <to_id> [--dry-run]` | Apply code changes based on doc diff *(experimental)* |
| `gc [--keep-last N] [--keep-days D] [--dry-run]` | Clean up old snapshots |
| `pin <snapshot_id>` | Mark snapshot as permanent |
| `unpin <snapshot_id>` | Remove pin from snapshot |

Snapshot IDs look like `YYYYMMDD-HHMMSS-<hash>` (example: `20260205-170300-a3f2c1`).
