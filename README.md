# sync-my-skills

[한국어](README.ko.md)

Sync your Claude Code skills across multiple machines using GitHub.

## Why

Claude Code skills live in local files (`~/.claude/skills/`) and don't sync across devices through your account. If you use Claude Code on multiple machines — work, home, laptop — you've probably wanted your custom skills everywhere without manually copying files.

This plugin automates that: push skills to a private GitHub repo from one machine, pull them on another.

## Prerequisites

- [Claude Code](https://claude.ai/code)
- [GitHub CLI (`gh`)](https://cli.github.com/) — installed and authenticated
- Git

## Installation

Install via Claude Code plugin:

```
/plugin marketplace add kunsanglee/sync-my-skills
/plugin install sync-my-skills
```

Or install manually:

```bash
mkdir -p ~/.claude/skills/my-skills
curl -o ~/.claude/skills/my-skills/SKILL.md \
  https://raw.githubusercontent.com/kunsanglee/sync-my-skills/main/skills/my-skills/SKILL.md
```

## Usage

Run these commands inside Claude Code.

### Initial Setup

```
/my-skills setup
```

- Verifies your GitHub account via `gh` CLI
- Creates a **private** repo (or clones an existing one)
- Generates a config file (`~/.my-skills.yml`)
- If skills already exist in the repo (from another machine), offers to link them

### Push Skills (Local → GitHub)

```
/my-skills push                            # Push all linked user-level skills
/my-skills push memo                       # Push a specific skill
/my-skills push memo recap                 # Push multiple skills
/my-skills push backend-init -p            # Push project-level skill (auto-detect project)
/my-skills push backend-init -p retrievo   # Push to a specific project
```

The first push copies the skill to the repo and replaces the original with a symlink. Subsequent pushes just commit and push changes.

### Pull Skills (GitHub → Local)

```
/my-skills pull                            # Pull all updates + offer to link new skills
/my-skills pull memo                       # Pull a specific skill
/my-skills pull memo recap                 # Pull multiple skills
/my-skills pull -p retrievo                # Show retrievo's skills, select which to pull
/my-skills pull backend-init -p retrievo   # Pull a specific project skill directly
```

### List Skills

```
/my-skills list                   # Show all skills (user + project level)
/my-skills list -p retrievo       # Show skills for a specific project
```

Example output:
```
Repo: username/my-skills (private)

── User Level ──
[linked]     memo            Create study notes from conversations
[linked]     recap           Summarize session work
[local only] backend-init    Spring Boot project initializer
[other]      skill-creator   Create and improve skills (via plugin)

── Project Level ──
  retrievo/
    [linked]     backend-init  Spring Boot + Kotlin DDD project init
    [remote]     claude-api    Build apps with Claude API
```

### Unlink Skills

```
/my-skills unlink memo                  # Unlink a user-level skill
/my-skills unlink memo recap            # Unlink multiple skills
/my-skills unlink backend-init -p       # Unlink a project-level skill
```

Removes the symlink and restores the original from backup. The skill stays in the repo — only the local link is removed.

## How It Works

```
~/.claude/skills/memo  →  (symlink)  →  ~/.my-skills-repo/user/memo
                                              ↕ (git push/pull)
                                        GitHub: username/my-skills
                                              ↕ (git push/pull)
[Other machine] ~/.my-skills-repo/user/memo  ←  (symlink)  ←  ~/.claude/skills/memo
```

1. When you push a skill, it's copied to `~/.my-skills-repo/user/` (or `projects/<name>/` for project-level) and the original is replaced with a symlink.
2. After that, editing the skill directly modifies the file in the repo directory (through the symlink).
3. `/my-skills push` commits and pushes changes to GitHub. `/my-skills pull` on another machine fetches them.

## Repo Structure

```
~/.my-skills-repo/
├── user/                       ← user-level skills
│   ├── memo/SKILL.md
│   └── recap/SKILL.md
└── projects/                   ← project-level skills (by project name)
    ├── retrievo/
    │   └── backend-init/SKILL.md
    └── execute/
        └── frontend-init/SKILL.md
```

## User Level vs Project Level

| Scope | Path | Applies to |
|-------|------|------------|
| User level | `~/.claude/skills/` | All projects |
| Project level | `{project}/.claude/skills/` | That project only |

- Default scope is user level
- Use `--project` (`-p`) flag for project-level skills

## Status Labels

| Label | Meaning |
|-------|---------|
| `[linked]` | Symlink connected to repo — synced |
| `[remote]` | Exists in repo but not linked on this machine |
| `[local only]` | Exists locally but not in repo |
| `[other]` | Symlink pointing elsewhere (e.g. plugin-installed) |

## FAQ

**Q: Will my existing skills be deleted?**
A: No. Original skills are backed up to `~/.my-skills-backup/` before being replaced with symlinks.

**Q: What happens when I edit a skill?**
A: The symlink points to the repo directory, so edits go directly to the repo. Just run `/my-skills push` to sync.

**Q: What if I edit the same skill on two machines?**
A: A Git conflict may occur. The tool will show the conflict and help you resolve it.

**Q: Is the repo public?**
A: No, it's created as **private** by default. Your personal skills stay private.

**Q: What if I push the same skill name from different projects?**
A: No conflict — project-level skills are stored under `projects/<project-name>/`, so the same skill name can exist in multiple projects.

## Feedback & Feature Requests

Found a bug or have an idea? Open a GitHub Issue.

- [Open an Issue](https://github.com/kunsanglee/sync-my-skills/issues)
- Bug reports, feature requests, and feedback are all welcome.

## Author

- [@kunsanglee](https://github.com/kunsanglee)
