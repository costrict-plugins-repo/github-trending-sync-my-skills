---
name: my-skills
description: "Sync and manage Claude Code skills across multiple machines using GitHub. Use this skill whenever the user says /my-skills, wants to sync skills between devices, back up their skills to GitHub, install skills from a remote repo, manage skill synchronization, or asks about sharing skills across machines. Trigger on phrases like 'sync my skills', 'push skills', 'pull skills', 'set up skill sync', 'list my remote skills'."
---

# my-skills — Claude Code Skill Sync Manager

Sync Claude Code skills across multiple machines using a personal GitHub repository. Manages individual skill-level symlinks so the user's existing `~/.claude/skills/` directory stays intact.

## Prerequisites

- `gh` CLI installed and authenticated (`gh auth status`)
- `git` installed

## Configuration

- **Config file**: `~/.my-skills.yml`
- **Local repo clone**: `~/.my-skills-repo/`

## Repo Structure

The repo separates user-level and project-level skills:

```
~/.my-skills-repo/
├── user/                          ← user level skills
│   ├── memo/SKILL.md
│   └── recap/SKILL.md
└── projects/                      ← project level skills (by project name)
    ├── retrievo/
    │   └── backend-init/SKILL.md
    └── execute/
        └── frontend-init/SKILL.md
```

**Symlink paths:**
- User level: `~/.claude/skills/<skill>` → `~/.my-skills-repo/user/<skill>`
- Project level: `{cwd}/.claude/skills/<skill>` → `~/.my-skills-repo/projects/<project>/<skill>`

## Commands

Parse the user's input to determine which command to run. The user will typically say `/my-skills <command> [args]`.

---

### `setup`

Initial setup: connect to GitHub, create or clone the repo, and configure.

**Steps:**

1. Verify `gh` is installed and authenticated:
   ```bash
   gh auth status
   ```
   If not authenticated, tell the user to run `! gh auth login` to authenticate.

2. Check if `~/.my-skills.yml` already exists. If so, read it and ask if the user wants to reconfigure.

3. Ask the user for the repo name (default: `my-skills`). The repo will be created as **private**.

4. Check if the repo already exists on GitHub:
   ```bash
   gh repo view <username>/my-skills --json name 2>/dev/null
   ```

5. If the repo exists, clone it:
   ```bash
   git clone https://github.com/<username>/my-skills.git ~/.my-skills-repo
   ```
   If it doesn't exist, create it:
   ```bash
   gh repo create my-skills --private --description "My Claude Code skills" --clone --gitignore=""
   ```
   Then move the cloned repo to `~/.my-skills-repo` if needed. Create the directory structure:
   ```bash
   mkdir -p ~/.my-skills-repo/user ~/.my-skills-repo/projects
   ```

6. Create `~/.my-skills.yml`:
   ```yaml
   repo: <username>/my-skills
   local_path: ~/.my-skills-repo
   linked_skills: []
   ```

7. If the cloned repo already contains skills (from another machine), list them and ask which ones to link:
   ```bash
   # User level skills
   ls -d ~/.my-skills-repo/user/*/SKILL.md 2>/dev/null | sed 's|.*/user/\(.*\)/SKILL.md|\1|'
   # Project level skills
   find ~/.my-skills-repo/projects -name SKILL.md 2>/dev/null
   ```
   For each selected user-level skill, create a symlink (see the Linking section below).

8. Report setup complete with a summary of what was configured.

---

### `push`

Push local skills to the remote repo.

**Usage:**
- `/my-skills push` — push all linked user-level skills
- `/my-skills push <skill-name>` — push a specific user-level skill
- `/my-skills push <skill1> <skill2>` — push multiple user-level skills
- `/my-skills push <skill-name> --project (-p)` — push a project-level skill (auto-detect project name from cwd, confirm with user)
- `/my-skills push <skill-name> --project (-p) <project-name>` — push a project-level skill to a specific project name

**Steps:**

1. Read `~/.my-skills.yml` to get config. If not found, tell the user to run `setup` first.

2. Determine scope and source directory:
   - **User level** (default): source is `~/.claude/skills/`, repo target is `~/.my-skills-repo/user/`
   - **Project level** (`--project (-p)`):
     - Source is `{cwd}/.claude/skills/`
     - Detect project name: `basename $(git rev-parse --show-toplevel 2>/dev/null || pwd)`
     - If project name is not specified via argument, show the detected name and ask the user to confirm
     - Repo target is `~/.my-skills-repo/projects/<project-name>/`

3. If no skill names specified (user level only), push all currently linked skills:
   - For each skill in `linked_skills`, changes are already in the repo directory via symlink. Just commit and push.

4. If skill names are specified:
   - For each skill, check if it exists in the source directory
   - If the skill is not yet in the repo, copy it to the appropriate repo path:
     ```bash
     # User level
     cp -r <source>/<skill-name> ~/.my-skills-repo/user/<skill-name>
     # Project level
     mkdir -p ~/.my-skills-repo/projects/<project-name>
     cp -r <source>/<skill-name> ~/.my-skills-repo/projects/<project-name>/<skill-name>
     ```
   - Then replace the original with a symlink (see Linking section)
   - For user-level skills, add to `linked_skills` in `~/.my-skills.yml`

5. Commit and push:
   ```bash
   cd ~/.my-skills-repo
   git add -A
   git commit -m "Update skills: <skill-names>"
   git push
   ```

6. Report which skills were pushed.

---

### `pull`

Pull skills from the remote repo to this machine.

**Usage:**
- `/my-skills pull` — pull all updates for linked user-level skills + show new ones
- `/my-skills pull <skill-name>` — pull and link a specific user-level skill
- `/my-skills pull <skill1> <skill2>` — pull and link multiple user-level skills
- `/my-skills pull <skill-name> --project (-p)` — show project list from repo, let user select, then pull
- `/my-skills pull <skill-name> --project (-p) <project-name>` — pull a project-level skill directly

**Steps:**

1. Read `~/.my-skills.yml`. If not found, tell the user to run `setup` first.

2. Pull latest from remote:
   ```bash
   cd ~/.my-skills-repo
   git pull
   ```

3. Determine scope and target directory:
   - **User level** (default): target is `~/.claude/skills/`, repo source is `~/.my-skills-repo/user/`
   - **Project level** (`--project (-p)`):
     - Target is `{cwd}/.claude/skills/`
     - If project name is not specified, list available projects from the repo and let the user select:
       ```bash
       ls ~/.my-skills-repo/projects/
       ```
     - Repo source is `~/.my-skills-repo/projects/<project-name>/`

4. If no skill names specified (user level):
   - Update all currently linked skills (symlinks already point to repo, so git pull is enough)
   - List any new skills in `~/.my-skills-repo/user/` that aren't linked yet, and ask if the user wants to link them

5. If skill names are specified:
   - For each skill, verify it exists in the appropriate repo path
   - Create symlink in the target directory (see Linking section)
   - For user-level skills, add to `linked_skills` in `~/.my-skills.yml`

6. Report which skills were pulled/linked.

---

### `list`

Show status of all skills (local and remote), separated by scope.

**Usage:**
- `/my-skills list` — show all user-level and project-level skills
- `/my-skills list --project (-p) <project-name>` — show skills for a specific project only

**Steps:**

1. Read `~/.my-skills.yml`. If not found, tell the user to run `setup` first.

2. Pull latest repo info (fetch only, don't merge):
   ```bash
   cd ~/.my-skills-repo
   git fetch
   ```

3. Collect information (use `bash -c '...'` for complex shell scripts to avoid zsh compatibility issues):
   - User-level skills in `~/.my-skills-repo/user/`
   - User-level skills linked locally (symlinks in `~/.claude/skills/` pointing to `~/.my-skills-repo/user/`)
   - Local-only user-level skills (in `~/.claude/skills/`, not a symlink)
   - Other skills (symlinks pointing to somewhere other than `~/.my-skills-repo/`, e.g. plugin-installed skills)
   - Project-level skills in `~/.my-skills-repo/projects/` (grouped by project name)

   When checking symlinks, strip trailing slashes from paths before using `[ -L ]`. Use `readlink` to determine the symlink target.

4. Display as a table with these status labels:
   - `[linked]` — symlink points to `~/.my-skills-repo/`
   - `[remote]` — exists in repo but not linked on this machine
   - `[local only]` — exists locally but not in repo (regular directory, not a symlink)
   - `[other]` — symlink pointing to somewhere else (e.g. plugin-installed)

   ```
   Repo: <username>/my-skills (private)

   ── User Level ──
   [linked]     memo            Create study notes from conversations
   [linked]     recap           Summarize session work
   [local only] backend-init    (no description)
   [other]      skill-creator   Create and improve skills (via plugin)

   ── Project Level ──
     retrievo/
       [linked]     backend-init  Spring Boot + Kotlin DDD project init
       [remote]     claude-api    Build apps with Claude API
   ```

   To get skill descriptions, read each SKILL.md and extract the `description` from frontmatter.

5. If there are updates available (local behind remote), note them.

---

### `unlink`

Remove symlink and restore original skill from backup.

**Usage:**
- `/my-skills unlink <skill-name>` — unlink a user-level skill
- `/my-skills unlink <skill1> <skill2>` — unlink multiple skills
- `/my-skills unlink <skill-name> --project (-p)` — unlink a project-level skill

**Steps:**

1. Read `~/.my-skills.yml`. If not found, tell the user to run `setup` first.

2. For each skill:
   - Verify it's currently a symlink pointing to `~/.my-skills-repo/`
   - Remove the symlink
   - If a backup exists in `~/.my-skills-backup/<skill>`, restore it:
     ```bash
     mv ~/.my-skills-backup/<skill> <target>/<skill>
     ```
   - If no backup exists, copy from repo to make it a regular directory:
     ```bash
     cp -r ~/.my-skills-repo/user/<skill> <target>/<skill>
     ```
   - Remove from `linked_skills` in `~/.my-skills.yml` (for user-level)

3. The skill remains in the repo — only the local symlink is removed. The skill becomes a standalone local copy.

4. Report which skills were unlinked.

---

## Linking (Symlink Management)

When linking a skill, handle these cases carefully:

1. **Skill exists in target as a regular directory** (not yet linked):
   - Back up to `~/.my-skills-backup/` (outside the skills directory to avoid being recognized as a skill):
     ```bash
     mkdir -p ~/.my-skills-backup
     mv <target>/<skill> ~/.my-skills-backup/<skill>
     ```
   - Create symlink:
     - User level: `ln -s ~/.my-skills-repo/user/<skill> <target>/<skill>`
     - Project level: `ln -s ~/.my-skills-repo/projects/<project>/<skill> <target>/<skill>`
   - Tell the user the original was backed up to `~/.my-skills-backup/<skill>`

2. **Skill already linked** (symlink exists and points to repo):
   - Nothing to do, already synced

3. **Skill exists as a symlink to somewhere else**:
   - Warn the user and ask before replacing

4. **Skill doesn't exist in target**:
   - Just create the symlink

## Config File Format

`~/.my-skills.yml`:
```yaml
repo: kunsanglee/my-skills
local_path: ~/.my-skills-repo
linked_skills:
  - memo
  - recap
```

`linked_skills` tracks user-level linked skills only. Project-level links are determined by checking symlinks in the project's `.claude/skills/` directory.

## Error Handling

- If `gh` is not installed: "gh CLI is required. Install with: brew install gh"
- If not authenticated: "GitHub authentication required. Run: ! gh auth login"
- If config not found for push/pull/list: "Run /my-skills setup first."
- If git conflicts occur during pull: show the conflict and ask the user how to resolve
- If a skill name doesn't exist: list available skills and suggest the closest match

## Language

Communicate with the user in Korean (matching their preference), but keep technical terms (GitHub, symlink, push, pull, etc.) in English.
