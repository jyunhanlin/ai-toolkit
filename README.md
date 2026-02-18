# ai-toolkit

A collection of AI-powered development tools — skills, plugins, and utilities that enhance AI agents (Claude Code, Cursor, etc.) in real-world development workflows.

## Philosophy

- **Local-first**: Everything runs on your machine, no external services required
- **Lightweight**: Minimal setup, no heavy dependencies
- **Composable**: Each tool works independently, combine as needed

## Available Skills

| Skill | Description | Status |
|-------|-------------|--------|
| [local-doc-versioning](skills/local-doc-versioning/) | Git-like versioning for AI-generated feature docs — use only when Git isn't suitable for the files (e.g., ephemeral specs, temporary drafts) | v0.1.0 |

## Installation

### Via Skills CLI

```bash
# Install a specific skill
npx skills add jyunhanlin/ai-toolkit -s local-doc-versioning
```

### Manual

```bash
git clone https://github.com/jyunhanlin/ai-toolkit.git
cp -r ai-toolkit/skills/local-doc-versioning ~/.claude/skills/
```

Then restart Claude Code or run `/refresh-skills`.

## Repo Structure

```
ai-toolkit/
├── skills/                 # Claude Code skills
│   └── local-doc-versioning/
├── docs/                   # Design docs and plans
├── .github/                # CI/CD workflows
├── LICENSE
└── README.md
```

## Contributing

1. Fork the repo
2. Create a feature branch
3. Submit a PR with a clear description

## License

MIT
