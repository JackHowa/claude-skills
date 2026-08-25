# claude-skills

Personal slash commands and auto-triggered skills for Claude Code.

## Setup

Symlink the `commands` directory into `~/.claude/`:

```bash
ln -s ~/sites/claude-skills/commands ~/.claude/commands
```

Each skill under `skills/` is symlinked individually into `~/.claude/skills/`:

```bash
ln -s ~/sites/claude-skills/skills/<skill-name> ~/.claude/skills/<skill-name>
```

## Structure

```
commands/
  skill-name.md        # invoked explicitly as /skill-name
skills/
  skill-name/
    SKILL.md            # auto-triggered by Claude based on its description
```

## Writing a skill

Each `.md` file in `commands/` becomes a `/skill-name` slash command. The file is the prompt Claude receives when the skill is invoked. Use `$ARGUMENTS` to reference anything the user types after the slash command name.

Each directory under `skills/` with a `SKILL.md` is a skill Claude can invoke on its own based on the `description` in its frontmatter — no explicit slash command needed. Use this for workflows that should trigger from natural phrasing ("how are my PRs doing") rather than a fixed command name.
