# skillz2paybillz

A collection of [Claude Code](https://claude.com/claude-code) skills I've built
to (allegedly) pay the bills.

Each skill is defined by a **build prompt** -- a markdown file you paste into
Claude Code that generates the complete skill on your machine.

## Available Skills

| Skill | Description |
|-------|-------------|
| [mac-updater](./mac-updater/) | Mac system administrator and security auditor. Checks for outdated packages, security vulnerabilities, system health issues, and misconfigurations. |

## How to Install a Skill

1. Open the skill's directory (e.g. `mac-updater/`)
2. Open its build prompt (`build-*-skill.md`)
3. Copy the entire contents
4. Paste into Claude Code
5. Claude will generate the skill files in `~/.claude/skills/<skill-name>/`

After that, you can invoke the skill by name (e.g. `/mac-updater`) from any
Claude Code session.

## Why Build Prompts Instead of Direct Skill Files?

The build prompt is the source of truth. It:

- Documents exactly what the skill does and why
- Adapts to the user's machine during installation (detects installed tools,
  checks for version-specific behavior, etc.)
- Is easier to review, diff, and improve than a directory of opaque files
- Keeps the skill up to date with each iteration

You could extract the rendered skill files from a prompt, but they'd go stale
every time I improve the prompt. The prompt is always current.

## Warning

These skills are built for personal Macs. Do NOT run them on work-managed or
MDM-enrolled computers without checking with your IT department first. They may
flag, conflict with, or inadvertently disrupt managed configurations.

## License

MIT -- use, modify, share freely. No warranty.
