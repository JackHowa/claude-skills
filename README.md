# claude-skills

Personal slash command skills for Claude Code.

## Setup

Symlink the `commands` directory into `~/.claude/`:

```bash
ln -s ~/sites/claude-skills/commands ~/.claude/commands
```

## Structure

```
commands/
  skill-name.md   # invoked as /skill-name
```

## Writing a skill

Each `.md` file in `commands/` becomes a `/skill-name` slash command. The file is the prompt Claude receives when the skill is invoked. Use `$ARGUMENTS` to reference anything the user types after the slash command name.
