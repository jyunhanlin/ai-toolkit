# Experimental: plan & apply

> **Warning**: These commands rely on Claude's reasoning to derive code changes from doc diffs. Results may vary. Always review output carefully and use `--dry-run` before applying.

These commands use **stable IDs** in doc headings (e.g. `## REQ-001: Feature Name`) to track which sections changed between snapshots and derive corresponding code operations.

## Additional Configuration

These fields in `.agentdocs/config.json` are only used by `plan` and `apply`:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `doc_format.require_stable_ids` | boolean | `true` | Warn if docs lack stable section IDs |
| `doc_format.id_pattern` | string (regex) | `REQ-\\d+` | Pattern for stable IDs in doc headings |

## Additional Storage

When `apply` is used, operation logs are stored in `.agentdocs/applied/`:

```
.agentdocs/
  applied/                           # Operation logs from apply command
    <snapshot_id>.json
```

Setup creates this directory automatically when `apply` is first used.

## plan — Derive a Code Change Plan

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

## apply — Apply Code Changes

**Syntax**: `/local-doc-versioning apply <from_id> <to_id> [--dry-run]`

**Execution flow**:

1. Run the `plan` flow internally to get the change plan
2. For each operation in the plan:
   - **New**: generate the new code based on the doc requirements, use **Write** to create target files
   - **Modified**: use **Read** to load the existing target file, use **Edit** to apply minimal edits based on the doc changes
   - **Removed**: use **Read** to identify the code sections tied to deleted requirements, use **Edit** to remove them
3. If `--dry-run`: display what would change without writing any files. Show the planned edits for each target file.
4. If not dry-run:
   - Apply edits to target files
   - If an edit conflicts (target code has changed in unexpected ways), mark the conflict:
     ```
     <<<<<<< CURRENT
     ... existing code ...
     =======
     ... proposed change ...
     >>>>>>> PROPOSED (<requirement ID>)
     ```
   - Use **Bash**: `mkdir -p .agentdocs/applied` (if it doesn't exist)
   - Use **Write** to record all operations to `.agentdocs/applied/<to_id>.json` (see schema below)
5. Report:
   ```
   Applied changes: <from_id> → <to_id>
     <N> files modified
     <N> operations applied
     <N> conflicts (require manual review)
   ```

**Important**: Never silently overwrite code. Always prefer minimal, surgical edits. When in doubt, mark conflicts and ask the user.

**Error conditions**:
- Target files do not exist (for modify/remove) → report: "Target file `<path>` not found. Skipping operation `<id>`."
- Conflicts detected → mark conflict regions, do not silently overwrite
- All operations failed → report summary of failures

## Operation Log Schema (applied/<snapshot_id>.json)

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
