# cc-token-optimizer-skill

A Claude Code skill that audits and actively compacts all Claude memory files on your machine, reducing context size for faster, cheaper sessions.

## What it does

Running `/token-optimizer` performs 6 checks in sequence, each tracked as a task:

| # | Check | Action |
|---|-------|--------|
| 1 | `.claudeignore` | Creates it if missing; fills in missing patterns |
| 2 | `~/.claude/settings.json` | Ensures `model: opusplan` and `effortLevel: high` |
| 3 | **Memory compaction** | Scans every Claude memory file on your machine → backs them up → spawns parallel agents to compress each file without losing meaning |
| 4 | MCP server audit | Flags MCPs that can be replaced with CLI tools |
| 5 | Subagent model audit | Recommends `haiku/sonnet/opus` per agent role |
| 6 | Context usage | Warns if session context is approaching 60% |

### Memory compaction in detail (check #3)

Scope — files included in the scan:

- `~/.claude/CLAUDE.md` (global instructions)
- All project `CLAUDE.md` files (up to depth 6, excluding `node_modules` and worktrees)
- All `~/.claude/projects/*/memory/MEMORY.md` index files
- All `~/.claude/projects/*/memory/*.md` individual memory entries

Scope — always excluded:

- `~/.claude/skills/**` (skill definitions, not your memory)
- `**/node_modules/**`
- `**/.claude/worktrees/**`
- `~/.claude/file-history/**` and `~/.claude/backups/**`

What the compaction agents do:

- Keep every semantic fact (rules, paths, commands, names, dates, URLs, env var names)
- Keep YAML frontmatter and `Why:` / `How to apply:` fields
- Remove repeated explanations, redundant examples, filler phrases
- Rewrite prose as bullet points where possible
- Write the compacted content back to the original path
- Report `BEFORE → AFTER` line counts and reduction %

Safety: a full backup is created at `~/.claude/backups/memory-compact-<timestamp>/` before any file is touched. One-line restore: `cp -r ~/.claude/backups/memory-compact-<timestamp>/* /`

## Installation

### Requirements

- [Claude Code](https://claude.ai/code) installed and authenticated
- Skills directory at `~/.claude/skills/` (Claude Code creates this automatically)

### Steps

1. Create the skill directory:

```bash
mkdir -p ~/.claude/skills/token-optimizer
```

2. Download the skill file:

```bash
curl -o ~/.claude/skills/token-optimizer/skill.md \
  https://raw.githubusercontent.com/dswf65411-new/cc-token-optimizer-skill/main/skill.md
```

Or clone and symlink if you want to stay up to date:

```bash
git clone https://github.com/dswf65411-new/cc-token-optimizer-skill.git ~/codes/cc-token-optimizer-skill
mkdir -p ~/.claude/skills/token-optimizer
ln -s ~/codes/cc-token-optimizer-skill/skill.md ~/.claude/skills/token-optimizer/skill.md
```

3. Verify Claude Code can see the skill — open a new session and check:

```
/skills
```

You should see `token-optimizer` in the list.

## Usage

In any Claude Code session, type:

```
/token-optimizer
```

Or trigger it with natural language:

- "幫我優化 token 設定"
- "檢查 Claude Code 設定"
- "我 token 消耗太快"
- "幫我做 token 健檢"

Claude will create a task list for all 6 checks and work through them one by one, asking for confirmation before making any changes that could be destructive (e.g., CLAUDE.md edits, memory compaction when > 30 files are involved).

## When to run

- When starting a new project
- When sessions feel sluggish or context fills up faster than expected
- Periodically (e.g., monthly) to keep memory files lean

## File structure

```
cc-token-optimizer-skill/
├── skill.md      # The skill definition loaded by Claude Code
└── README.md     # This file
```

## License

MIT
