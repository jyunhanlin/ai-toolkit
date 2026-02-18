# Data Schemas & Configuration

## Configuration

Configuration lives in `.agentdocs/config.json`. See `assets/config.json` for defaults.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `source_root` | string | `.claude/feature-docs` | Directory to read feature docs from |
| `retention.keep_last` | integer | `10` | Keep at least this many recent snapshots |
| `retention.keep_days` | integer | `7` | Also keep snapshots from the last N days |
| `retention.pins` | string[] | `[]` | Snapshot IDs exempt from GC |

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
```

## manifest.json

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
      "size_bytes": 2048
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
