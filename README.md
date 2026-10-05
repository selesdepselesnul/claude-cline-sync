# claude-cline-sync

Use your Claude Code setup in [Cline CLI](https://github.com/cline/cline): global rules, skills, subagents and MCP servers.

Handy when you switch to Cline (for example with a different model provider) and want the same agents and conventions there.

Claude Code stays the source of truth.
Keep editing things under `~/.claude`; the Cline side is linked or generated from it.

Sister project for opencode: [claude-opencode-sync](https://github.com/selesdepselesnul/claude-opencode-sync).

## What is shared and how

| Claude Code | Cline CLI | How |
| --- | --- | --- |
| `~/.claude/CLAUDE.md` | `~/.cline/rules/AGENTS.md` | Symlink |
| `~/.claude/skills/<name>/` | `~/.cline/skills/<name>/` | One symlink per skill |
| `~/.claude/agents/<name>.md` | `~/.cline/agents/<name>.yaml` | Generated copy |
| `mcpServers` in `~/.claude.json` | `~/.cline/data/settings/cline_mcp_settings.json` | Merged |

Cline reads nothing from `~/.claude` on its own, so all four need bridging.
A few details that shaped the design, found by reading Cline's loaders:

- Rule directories skip symlinked files, except a file named exactly `AGENTS.md`, which is why the rules link has that name.
- Skill discovery follows symlinked skill folders.
  Linking per skill (not the whole folder) means skills you install through Cline land in `~/.cline/skills` and never inside `~/.claude/skills`.
- Agents are only loaded from `.yml` / `.yaml` files and symlinks are skipped, so agents are written as real files.
  The content format (YAML frontmatter plus a markdown body used as the system prompt) is the same as Claude Code's.
- MCP server entries use the same shape as Claude Code's, so they are copied as is.

## Requirements

- Python 3.8+
- PyYAML (`pip install -r requirements.txt`)
- Cline CLI 3.x (tested on 3.0.68)

## Usage

```sh
git clone https://github.com/selesdepselesnul/claude-cline-sync.git
cd claude-cline-sync
pip install -r requirements.txt

python3 sync.py --dry-run   # preview what would change
python3 sync.py             # apply
```

In Cline:

- each agent becomes a tool named `subagent_<name>` (dashes turned into underscores), so ask for it by name, for example "use the bug hunter subagent on this repo"
- skills and rules load automatically

### Sync on every launch (optional)

Add a wrapper to `~/.zshrc` or `~/.bashrc` so every Cline launch syncs first.
It takes milliseconds and prints nothing unless something changed.

```sh
cline() { python3 <path-to-repo>/sync.py --quiet; command cline "$@"; }
```

## How agents are converted

- `name`, `description` and the body are kept verbatim
- `model: inherit` and `color` are dropped, Cline ignores them anyway
- a concrete `model` or a `tools` list is skipped with a warning, since Claude Code and Cline name models and tools differently

## Safety

- Generated agent files start with a marker comment.
  Only marked files are ever overwritten or deleted, so agents you wrote by hand for Cline are left alone.
- Existing files or links that don't match what the script expects are never replaced, only reported.
- Stale skill links are removed only when they point into `~/.claude/skills`.
- The MCP merge keeps all other servers and keys, saves `cline_mcp_settings.json.bak` before writing, and skips the run if Cline currently holds a lock on the file.
- MCP env vars and headers are copied as is, so any secrets in `~/.claude.json` also end up in Cline's settings on the same machine.
- Nothing under `~/.claude` is ever modified.
- The script is idempotent; running it twice in a row reports "already in sync".

## Known gaps

- Claude.ai connectors (Gmail, Drive, Notion and other account level integrations) belong to your Claude account and do not exist in Cline.
  Only MCP servers configured in `~/.claude.json` are carried over.
- Agent prompts often name Claude Code tools (`Agent`, `Task`, `Skill`, `WebFetch`).
  Cline has its own equivalents and models usually map them, but prompts are not rewritten.
- Claude Code auto memory (`~/.claude/projects/*/memory/`) is not shared.
- Project level `.claude/` folders are not synced, only the global ones in your home directory.

## License

MIT
