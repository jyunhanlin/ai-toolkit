---
name: local-doc-versioning
version: 0.1.0
description: >
  Git-like versioning for local feature documents.
  Save snapshots, recall history, diff changes, and apply incremental code updates
  without polluting Git history. Perfect for Claude-generated docs with short lifecycle.
invocation:
  mode: explicit
categories:
  - version-control
  - documentation
  - context-management
---

# Local Doc Versioning

Git-like version control for AI-generated feature documents that stay local and never touch your Git history.

## Overview

This skill provides local, Git-like version control for AI-generated feature documents. It creates snapshots of your feature docs, lets you diff and recall them, and can derive minimal code patches from doc changes.

Use it when you want to:

- Track versions of Claude-generated docs without committing to Git
- Recall previous documentation states for context replay
- Apply incremental code changes based on doc diffs (add/modify/delete)
- Automatically clean up old versions

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

1. Read the template config from the skill's `template/config.json`
2. Create the directory structure:
   ```
   .agentdocs/
   .agentdocs/snapshots/
   .agentdocs/applied/
   ```
3. Copy the template config to `.agentdocs/config.json`
4. Check if `.gitignore` exists in the project root:
   - If yes: read it. If `.agentdocs/` is not listed, append a newline and `.agentdocs/` to it.
   - If no: create `.gitignore` containing `.agentdocs/`
5. Confirm `source_root` (default `.claude/feature-docs`) exists. If not, create it.
6. Report to the user:
   ```
   Initialized local-doc-versioning:
     Store: .agentdocs/
     Source: .claude/feature-docs/
     Config: .agentdocs/config.json
     .gitignore updated
   ```

## Commands

### save — Create a Snapshot

**Syntax**: `/local-doc-versioning save [feature_key] [--message "description"]`

**Execution flow**:

1. Load config from `.agentdocs/config.json`
2. Validate `feature_key` if provided (lowercase alphanumeric + hyphens, 1–50 chars). If invalid, report error and stop.
3. Use Glob to list all files under `source_root` (recursively, excluding hidden files)
4. If no files found, report error: "No files found under `<source_root>`. Nothing to snapshot." and stop.
5. For each file:
   - Read the file content
   - Compute SHA-256 hash of the content
   - Record path (relative to `source_root`), hash, byte size, and last modified time
6. Generate the manifest JSON (see Data Schemas below) with all fields except `snapshot_id`:
   - Set `feature_key` from the positional argument (or null if not provided)
   - Set `summary` from the `--message` flag value (or null if not provided)
   - Set `created_at` to current UTC time
   - Set `source_root` from config
7. Compute `snapshot_id`: take current UTC timestamp formatted as `YYYYMMDD-HHMMSS`, append `-`, append first 6 hex chars of SHA-256 of the concatenated file hashes (sorted by path)
8. Create directory `.agentdocs/snapshots/<snapshot_id>/files/`
9. Copy each source file into `.agentdocs/snapshots/<snapshot_id>/files/` preserving relative paths
10. Write the complete manifest (now including `snapshot_id`) to `.agentdocs/snapshots/<snapshot_id>/manifest.json`
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
- `.agentdocs/` not writable → report permission error

---

### list — Show Snapshots

**Syntax**: `/local-doc-versioning list [--limit N]`

**Execution flow**:

1. Load config from `.agentdocs/config.json`
2. Use Glob to find all `manifest.json` files under `.agentdocs/snapshots/`
3. Read each manifest and extract: `snapshot_id`, `created_at`, `feature_key`, file count, total size
4. Check which snapshot IDs appear in `config.retention.pins`
5. Sort by `created_at` descending (newest first)
6. If `--limit N` specified, show only the first N entries
7. Display as a table:
   ```
   Recent snapshots:
     [pin] <snapshot_id>  [<feature_key>]  <N> files  <size>
     ...
   ```
   Use `(pinned)` marker for pinned snapshots.

**Error conditions**:
- No snapshots exist → report: "No snapshots found. Use `save` to create one."

---

### recall — Load a Previous Snapshot

**Syntax**: `/local-doc-versioning recall <snapshot_id>`

**Execution flow**:

1. Validate that `.agentdocs/snapshots/<snapshot_id>/manifest.json` exists. If not, report error: "Snapshot `<snapshot_id>` not found." and stop.
2. Read the manifest
3. Display snapshot summary:
   ```
   Snapshot: <snapshot_id>
   Feature: <feature_key>
   Created: <created_at>
   Message: <summary>

   Files:
     - <path> (<size>)
     ...
   ```
4. For each file in the snapshot, read it from `.agentdocs/snapshots/<snapshot_id>/files/<path>` and present the content to the user
5. Ask the user how they want to proceed:
   - **View only**: just display the content (default)
   - **Restore**: copy snapshot files back to `source_root`, overwriting current versions

**Error conditions**:
- Snapshot ID not found → report error with suggestion to run `list`
- Snapshot files missing/corrupted → report which files are missing

---

### diff — Compare Two Snapshots

**Syntax**: `/local-doc-versioning diff <from_id> <to_id>`

**Execution flow**:

1. Validate both snapshot IDs exist. If either is missing, report error and stop.
2. Read both manifests
3. Compare file lists:
   - **Added**: files in `to` but not in `from` (by relative path)
   - **Deleted**: files in `from` but not in `to`
   - **Modified**: files in both where SHA-256 hashes differ
   - **Unchanged**: files in both where hashes match
4. For each modified file:
   - Read both versions from their snapshot directories
   - Generate a unified diff (use Bash `diff -u` or produce the diff inline)
5. Display:
   ```
   Diff: <from_id> → <to_id>

   Files changed:
     A <path>            (new file, <N> lines)
     D <path>            (deleted)
     M <path>            (+<added> lines, -<removed> lines)

   Summary:
     - <natural language description of changes>
   ```

**Error conditions**:
- Either snapshot not found → report which one is missing
- Same snapshot ID for both → report: "Cannot diff a snapshot with itself."

---

### plan — Derive a Code Change Plan (Experimental)

> **Experimental**: This command relies on Claude's reasoning to derive code changes from doc diffs. Results may vary. Always review the output carefully.

**Syntax**: `/local-doc-versioning plan <from_id> <to_id>`

**Execution flow**:

1. Run the `diff` flow internally to get the list of changes
2. For each changed document, analyze the content:
   - Identify sections by stable IDs (matching `doc_format.id_pattern` from config)
   - Categorize changes per section: new section, modified section, deleted section
3. Check `.agentdocs/applied/` for any previous operation logs related to these snapshots
4. Produce a structured change plan:
   ```
   Change plan: <from_id> → <to_id>

   New operations:
     + <ID>: <description of what code to add>

   Modified operations:
     ~ <ID>: <description of what code to update>

   Removed operations:
     - <ID>: <description of what code to remove/revert>
   ```
5. Ask the user: **Preview patches**, **Apply**, or **Cancel**

**Example**: Given a doc that changed between two snapshots:

Before (snapshot A):
```markdown
## REQ-001: User Login
Users log in with email and password.

## REQ-002: Session Management
Sessions expire after 24 hours.
```

After (snapshot B):
```markdown
## REQ-001: User Login
Users log in with email and password. Add OAuth support.

## REQ-002: Session Management
Sessions expire after 1 hour (changed from 24).

## REQ-003: Audit Logging
Log all authentication events.
```

Expected plan output:
```
Change plan: <A> → <B>

Modified operations:
  ~ REQ-001: Update login to add OAuth support
  ~ REQ-002: Change session expiry from 24h to 1h

New operations:
  + REQ-003: Implement audit logging for auth events
```

**Error conditions**:
- Same as `diff` errors
- No stable IDs found in docs and `require_stable_ids` is true → warn: "No stable IDs found. Enable `require_stable_ids: false` in config or add IDs like `REQ-001` to doc headings."

---

### apply — Apply Code Changes (Experimental)

> **Experimental**: This command relies on Claude's reasoning to derive and apply code changes from doc diffs. Results may vary. Always use `--dry-run` first.

**Syntax**: `/local-doc-versioning apply <from_id> <to_id> [--dry-run]`

**Execution flow**:

1. Run the `plan` flow internally to get the change plan
2. For each operation in the plan:
   - **New**: generate the new code based on the doc requirements, write to target files
   - **Modified**: read the existing target file, apply minimal edits based on the doc changes
   - **Removed**: identify the code sections tied to deleted requirements, remove them
3. If `--dry-run`: display what would change without writing any files. Show the planned edits for each target file.
4. If not dry-run:
   - Apply edits to target files using the Edit tool
   - If an edit conflicts (target code has changed in unexpected ways), mark the conflict:
     ```
     <<<<<<< CURRENT
     ... existing code ...
     =======
     ... proposed change ...
     >>>>>>> PROPOSED (<requirement ID>)
     ```
   - Record all operations to `.agentdocs/applied/<to_id>.json` (see Data Schemas)
5. Report:
   ```
   Applied changes: <from_id> → <to_id>
     <N> files modified
     <N> operations applied
     <N> conflicts (require manual review)
   ```

**Error conditions**:
- Target files do not exist (for modify/remove) → report: "Target file `<path>` not found. Skipping operation `<id>`."
- Conflicts detected → mark conflict regions, do not silently overwrite
- All operations failed → report summary of failures

**Important**: Never silently overwrite code. Always prefer minimal, surgical edits. When in doubt, mark conflicts and ask the user.

---

### gc — Garbage Collection

**Syntax**: `/local-doc-versioning gc [--keep-last N] [--keep-days D] [--dry-run]`

**Execution flow**:

1. Load config. Use provided flags or fall back to `retention.keep_last` and `retention.keep_days` from config.
2. List all snapshots (same as `list` flow)
3. Determine which snapshots to keep (union of all rules):
   - The most recent N snapshots (by `created_at`)
   - Any snapshot created within the last D days
   - Any snapshot whose ID is in `config.retention.pins`
4. Mark all other snapshots for deletion
5. If `--dry-run`:
   ```
   GC preview:
     Keep: <N> snapshots (<breakdown>)
     Delete: <N> snapshots (~<total size>)

     Snapshots to delete:
       <snapshot_id>  [<feature_key>]  <created_at>
       ...
   ```
6. If not dry-run:
   - Delete each marked snapshot directory under `.agentdocs/snapshots/`
   - Delete corresponding operation logs under `.agentdocs/applied/` if they exist
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
2. Load config
3. If `snapshot_id` is already in `retention.pins`, report: "Snapshot `<snapshot_id>` is already pinned." and stop.
4. Add `snapshot_id` to `retention.pins` array
5. Write updated config to `.agentdocs/config.json`
6. Report: "Pinned snapshot `<snapshot_id>`. It will not be deleted by GC."

---

### unpin — Remove Pin from Snapshot

**Syntax**: `/local-doc-versioning unpin <snapshot_id>`

**Execution flow**:

1. Validate snapshot exists. If not, report error and stop.
2. Load config
3. If `snapshot_id` is not in `retention.pins`, report: "Snapshot `<snapshot_id>` is not pinned." and stop.
4. Remove `snapshot_id` from `retention.pins` array
5. Write updated config to `.agentdocs/config.json`
6. Report: "Unpinned snapshot `<snapshot_id>`. It may now be deleted by GC."

## Configuration

Configuration lives in `.agentdocs/config.json`. See `template/config.json` for defaults.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `source_root` | string | `.claude/feature-docs` | Directory to read feature docs from |
| `store_root` | string | `.agentdocs` | Directory for snapshot storage |
| `retention.keep_last` | integer | `10` | Keep at least this many recent snapshots |
| `retention.keep_days` | integer | `7` | Also keep snapshots from the last N days |
| `retention.pins` | string[] | `[]` | Snapshot IDs exempt from GC |
| `doc_format.require_stable_ids` | boolean | `true` | Warn if docs lack stable section IDs |
| `doc_format.id_pattern` | string (regex) | `REQ-\\d+` | Pattern for stable IDs in doc headings |

## Storage Structure

```
<project>/
  .claude/feature-docs/               # Source docs (configurable)
  .agentdocs/                          # Local-only version store (gitignored)
    config.json                        # Configuration
    snapshots/
      <snapshot_id>/
        manifest.json                  # Snapshot metadata
        files/                         # Snapshot file copies
          auth-flow.md
          api-spec.md
    applied/                           # Operation logs from apply command
      <snapshot_id>.json
```

## Data Schemas

### manifest.json

```json
{
  "snapshot_id": "20260205-170300-a3f2c1",
  "created_at": "2026-02-05T17:03:00Z",
  "source_root": ".claude/feature-docs",
  "feature_key": "user-auth",
  "summary": "Added OAuth flow and session management",
  "files": [
    {
      "path": "auth-flow.md",
      "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
      "size_bytes": 2048,
      "modified_at": "2026-02-05T17:02:55Z"
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `snapshot_id` | string | yes | Unique snapshot identifier |
| `created_at` | string (ISO 8601) | yes | When the snapshot was created |
| `source_root` | string | yes | Source directory at time of snapshot |
| `feature_key` | string | no | User-provided feature label |
| `summary` | string | no | User-provided description (from `--message`) |
| `files` | array | yes | List of snapshotted files |
| `files[].path` | string | yes | Relative path from source_root |
| `files[].sha256` | string | yes | Full SHA-256 hex digest of file content |
| `files[].size_bytes` | integer | yes | File size in bytes |
| `files[].modified_at` | string (ISO 8601) | yes | File's last modified time |

### Operation Log (applied/<snapshot_id>.json)

```json
{
  "from_snapshot": "20260204-093000-b1e4d2",
  "to_snapshot": "20260205-170300-a3f2c1",
  "applied_at": "2026-02-05T17:05:00Z",
  "operations": [
    {
      "operation_id": "op-001",
      "type": "add",
      "doc_refs": ["auth-flow.md#REQ-042"],
      "target_files": ["src/auth/oauth.ts"],
      "description": "Implement OAuth callback handler",
      "status": "applied"
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from_snapshot` | string | yes | Source snapshot ID |
| `to_snapshot` | string | yes | Target snapshot ID |
| `applied_at` | string (ISO 8601) | yes | When operations were applied |
| `operations` | array | yes | List of operations |
| `operations[].operation_id` | string | yes | Unique operation identifier (auto-generated) |
| `operations[].type` | string | yes | One of: `add`, `modify`, `remove` |
| `operations[].doc_refs` | string[] | yes | Document sections this operation relates to |
| `operations[].target_files` | string[] | yes | Code files affected |
| `operations[].description` | string | yes | What this operation does |
| `operations[].status` | string | yes | One of: `applied`, `conflict`, `skipped` |

## Guidelines

- **Use stable IDs in docs** for reliable diff tracking:
  ```markdown
  ## REQ-001: OAuth Implementation
  ## REQ-002: Session Management
  ```
- **Save snapshots frequently**: after each major doc update, before significant edits, at session end
- **Pin important versions**: feature completion, demos, release milestones
- **Run GC regularly**: `/local-doc-versioning gc` weekly to control disk usage
- **Always review plans before apply**: run `plan` first, inspect operations, then `apply`
- **All artifacts stay local** in `.agentdocs/` — this skill never writes to Git or suggests commits
- **Prefer minimal patches** over full-file rewrites when applying changes
- **On conflicts, mark and ask** — never silently overwrite code

## Troubleshooting

### Snapshots not being created

1. Verify `source_root` exists (default: `.claude/feature-docs/`)
2. Verify there are files under `source_root`
3. Verify `.agentdocs/` is writable

### Apply fails with conflicts

1. Conflict regions are marked with `<<<<<<<` / `=======` / `>>>>>>>` markers
2. Manually review and resolve those sections
3. Re-run `apply` if needed

### Storage growing too large

Run garbage collection:
```
/local-doc-versioning gc --keep-last 5 --keep-days 3
```

### Snapshot not found

Run `/local-doc-versioning list` to see available snapshots. Snapshot IDs are case-sensitive and must match exactly.
