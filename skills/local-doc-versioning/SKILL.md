---
name: local-doc-versioning
description: >
  Git-like versioning for local feature documents. Save snapshots, recall
  history, diff changes, and restore previous versions without polluting Git
  history. Use when user says "save a snapshot", "version my docs", "diff my
  documents", "restore previous version", "track doc changes", or needs to
  manage Claude-generated docs with short lifecycle. Do NOT use for files that
  should be committed to Git.
license: MIT
metadata:
  version: 0.1.0
  categories: version-control, documentation, context-management
---

# Local Doc Versioning

Git-like version control for AI-generated feature documents that stay local and never touch your Git history.

## Overview

**Use it when** docs are short-lived feature specs, experiments, or drafts you do not want in Git history.

**Do not use it when** docs are long-lived, formal specs that must be code-reviewed and checked into Git.

## Conventions

**Snapshot ID format**: `YYYYMMDD-HHMMSS-<hash>`
- Timestamp portion uses the current UTC time
- Hash is the first 6 hex characters of the SHA-256 digest of the concatenated file content hashes (sorted by relative path for determinism). This means identical file content at different times produces the same hash suffix.
- Example: `20260205-170300-a3f2c1`

**Feature key format**: Lowercase alphanumeric characters and hyphens only, 1–50 characters.
- Valid: `user-auth`, `payment-flow-v2`, `api-redesign`
- Invalid: `User Auth`, `payment_flow`, `a-very-long-feature-key-that-exceeds-fifty-characters-limit`

**Timestamp format**: ISO 8601 UTC, e.g. `2026-02-05T17:03:00Z`

## Setup and Auto-Initialization

On first invocation of any command, check whether `.agentdocs/` exists. If it does not, run this initialization flow:

1. Use **Read** to load the template config from the skill's `assets/config.json`
2. Use **Bash** to create the directory structure:
   ```bash
   mkdir -p .agentdocs/snapshots
   ```
3. Use **Write** to copy the template config to `.agentdocs/config.json`
4. Check if `.gitignore` exists in the project root:
   - If yes: Use **Read** to load it. If `.agentdocs/` is not listed, use **Edit** to append `.agentdocs/` on a new line.
   - If no: Use **Write** to create `.gitignore` containing `.agentdocs/`
5. Confirm `source_root` (default `.claude/feature-docs`) exists. If not, use **Bash**: `mkdir -p .claude/feature-docs`
6. Report to the user:
   ```
   Initialized local-doc-versioning:
     Store: .agentdocs/
     Source: .claude/feature-docs/
     Config: .agentdocs/config.json
     .gitignore updated
   ```

## Commands

> **Important**: Before executing any command below, first check if `.agentdocs/` exists. If not, run the Setup and Auto-Initialization flow above.

### save — Create a Snapshot

**Syntax**: `/local-doc-versioning save [feature_key] [--message "description"]`

**Execution flow**:

1. Use **Read** to load config from `.agentdocs/config.json`
2. Validate `feature_key` if provided (lowercase alphanumeric + hyphens, 1–50 chars). If invalid, report error and stop.
3. Use **Glob** to list all files under `source_root` (recursively, excluding hidden files)
4. If no files found, report error: "No files found under `<source_root>`. Nothing to snapshot." and stop.
5. For each file, use **Read** to get the file content and keep it in memory. Then use **Bash** to compute SHA-256 and file size in one call:
   ```bash
   shasum -a 256 <file1> <file2> ... && wc -c <file1> <file2> ...
   ```
   Parse the output: `shasum` outputs `<hash>  <path>` per line; `wc -c` outputs `<size> <path>` per line (ignore the final `total` line if multiple files).
   Record each file's relative path (relative to `source_root`), sha256 hash, and byte size.
6. Generate the manifest JSON (see Data Schemas below) with all fields except `snapshot_id`:
   - Set `feature_key` from the positional argument (or null if not provided)
   - Set `summary` from the `--message` flag value (or null if not provided)
   - Set `created_at` to current UTC time
   - Set `source_root` from config
7. Compute `snapshot_id`: use **Bash** to get UTC timestamp (`date -u '+%Y%m%d-%H%M%S'`), then compute hash from concatenated file hashes sorted by path: `echo -n "<hash1><hash2>..." | shasum -a 256` and take first 6 hex chars. Format: `<timestamp>-<hash>`
8. Use **Bash**: `mkdir -p .agentdocs/snapshots/<snapshot_id>/files/`
9. For each source file: use **Write** to save the content (already in memory from step 5) to `.agentdocs/snapshots/<snapshot_id>/files/<relative_path>`
10. Use **Write** to save the complete manifest (now including `snapshot_id`) to `.agentdocs/snapshots/<snapshot_id>/manifest.json`
11. Report:
    ```
    Snapshot saved: <snapshot_id>
      Feature: <feature_key or "(none)">
      Files: <count> (<file list>)
      Size: <total size>
    ```

**Error conditions**:
- `source_root` does not exist → report error, suggest running setup
- No files in `source_root` → report error as above
- Invalid `feature_key` → report format requirements

---

### list — Show Snapshots

**Syntax**: `/local-doc-versioning list [--feature <key>] [--limit N]`

**Execution flow**:

1. Use **Read** to load config from `.agentdocs/config.json`
2. Use **Glob** to find all `manifest.json` files under `.agentdocs/snapshots/*/`
3. Use **Read** on each manifest and extract: `snapshot_id`, `created_at`, `feature_key`, file count, total size
4. If `--feature <key>` specified, filter to only snapshots matching that `feature_key`
5. Check which snapshot IDs appear in `config.retention.pins`
6. Sort by `created_at` descending (newest first)
7. If `--limit N` specified, show only the first N entries
8. Display as a table:
   ```
   Recent snapshots:
     (pinned) 20260205-170300-a3f2c1  [user-auth]  3 files  12.5 KB
             20260204-093000-b1e4d2  [user-auth]  2 files   8.1 KB
   ```

**Error conditions**:
- No snapshots exist → report: "No snapshots found. Use `save` to create one."
- No snapshots match `--feature` filter → report: "No snapshots found for feature `<key>`."

---

### recall — View or Restore a Previous Snapshot

**Syntax**: `/local-doc-versioning recall <snapshot_id> [--content] [--restore]`

**Execution flow**:

1. Validate that `.agentdocs/snapshots/<snapshot_id>/manifest.json` exists. If not, report error: "Snapshot `<snapshot_id>` not found." and stop.
2. Use **Read** to load the manifest
3. Display snapshot summary:
   ```
   Snapshot: <snapshot_id>
   Feature: <feature_key>
   Created: <created_at>
   Message: <summary>

   Files:
     - <path> (<size>)
     - ...

   Use --content to view file contents, --restore to restore files.
   ```
4. If `--content` flag is set: for each file in the snapshot, use **Read** to load from `.agentdocs/snapshots/<snapshot_id>/files/<path>` and present the full content to the user.
5. If `--restore` flag is set:
   - First, list which files will be overwritten and confirm with the user before proceeding.
   - For each file: use **Read** to get snapshot content, then **Write** to overwrite `<source_root>/<path>`
   - Report: "Restored <N> files to `<source_root>/` from snapshot `<snapshot_id>`."

**Error conditions**:
- Snapshot ID not found → report error with suggestion to run `list`
- Snapshot files missing/corrupted → report which files are missing

---

### diff — Compare Two Snapshots

**Syntax**: `/local-doc-versioning diff <from_id> <to_id>`

**Execution flow**:

1. Validate both snapshot IDs exist. If either is missing, report error and stop.
2. Use **Read** to load both manifests
3. Compare file lists:
   - **Added**: files in `to` but not in `from` (by relative path)
   - **Deleted**: files in `from` but not in `to`
   - **Modified**: files in both where SHA-256 hashes differ
   - **Unchanged**: files in both where hashes match
4. For modified files: use **Bash** `diff -u <from_file> <to_file>` to generate a unified diff (the files exist on disk in the snapshot directories, no need to Read them first).
5. Display:
   ```
   Diff: <from_id> → <to_id>

   Added:
     A <path>    (new file, <N> lines)

   Deleted:
     D <path>    (was <N> lines)

   Modified:
     M <path>    (+<added> -<removed> lines)
       <unified diff output>
   ```
   For added files, show `(new file, <N> lines)` — do not dump full content unless the user asks.
   For deleted files, show `(was <N> lines)` — do not dump deleted content.

**Error conditions**:
- Either snapshot not found → report which one is missing
- Same snapshot ID for both → report: "Cannot diff a snapshot with itself."

---

### gc — Garbage Collection

**Syntax**: `/local-doc-versioning gc [--keep-last N] [--keep-days D] [--dry-run]`

**Execution flow**:

1. Use **Read** to load config. Use provided flags or fall back to `retention.keep_last` and `retention.keep_days` from config.
2. Use **Bash**: `date -u '+%Y-%m-%dT%H:%M:%SZ'` to get current UTC time for age comparison.
3. List all snapshots (same as `list` flow)
4. Determine which snapshots to keep (union of all rules):
   - The most recent N snapshots (by `created_at`)
   - Any snapshot created within the last D days
   - Any snapshot whose ID is in `config.retention.pins`
5. Mark all other snapshots for deletion
6. If `--dry-run`:
   ```
   GC preview:
     Keep: <N> snapshots (<breakdown>)
     Delete: <N> snapshots (~<total size>)

     Snapshots to delete:
       <snapshot_id>  [<feature_key>]  <created_at>
       ...
   ```
7. If not dry-run:
   - Use **Bash** to delete each marked snapshot directory: `rm -rf .agentdocs/snapshots/<snapshot_id>`
   - Report:
     ```
     GC complete:
       Deleted: <N> snapshots (~<total size>)
       Remaining: <N> snapshots
     ```

**Error conditions**:
- No snapshots exist → report: "Nothing to clean up."
- All snapshots are protected → report: "All snapshots are retained by current rules. Nothing to delete."

---

### pin — Mark Snapshot as Permanent

**Syntax**: `/local-doc-versioning pin <snapshot_id>`

**Execution flow**:

1. Validate snapshot exists. If not, report error and stop.
2. Use **Read** to load config
3. If `snapshot_id` is already in `retention.pins`, report: "Snapshot `<snapshot_id>` is already pinned." and stop.
4. Add `snapshot_id` to `retention.pins` array (create new config object, do not mutate)
5. Use **Write** to save updated config to `.agentdocs/config.json`
6. Report: "Pinned snapshot `<snapshot_id>`. It will not be deleted by GC."

---

### unpin — Remove Pin from Snapshot

**Syntax**: `/local-doc-versioning unpin <snapshot_id>`

**Execution flow**:

1. Validate snapshot exists. If not, report error and stop.
2. Use **Read** to load config
3. If `snapshot_id` is not in `retention.pins`, report: "Snapshot `<snapshot_id>` is not pinned." and stop.
4. Remove `snapshot_id` from `retention.pins` array (create new config object, do not mutate)
5. Use **Write** to save updated config to `.agentdocs/config.json`
6. Report: "Unpinned snapshot `<snapshot_id>`. It may now be deleted by GC."

## Configuration, Storage & Data Schemas

For configuration fields, storage structure, and manifest.json schema, see [references/data-schemas.md](references/data-schemas.md).

Key defaults: source is `.claude/feature-docs/`, store is `.agentdocs/`, keeps last 10 snapshots and 7 days.

## Guidelines

- **Save snapshots frequently**: after each major doc update, before significant edits, at session end
- **Pin important versions**: feature completion, demos, release milestones
- **Run GC regularly**: `/local-doc-versioning gc` weekly to control disk usage
- **All artifacts stay local** in `.agentdocs/` — this skill never writes to Git or suggests commits

## Troubleshooting

### Snapshots not being created

1. Verify `source_root` exists (default: `.claude/feature-docs/`)
2. Verify there are files under `source_root`
3. Verify `.agentdocs/` is writable

### Storage growing too large

Run garbage collection:
```
/local-doc-versioning gc --keep-last 5 --keep-days 3
```

### Snapshot not found

Run `/local-doc-versioning list` to see available snapshots. Snapshot IDs are case-sensitive and must match exactly.

---

## Experimental: plan & apply

> These commands are experimental. Read the full specification before use.

For `plan` and `apply` commands that derive code changes from doc diffs, see [references/plan-apply.md](references/plan-apply.md).
