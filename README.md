# claude-tag-team-zcode-gemini

Install and usage guide for **zcode** (Z.ai GLM coding agent) and **agy**
(Google Antigravity coding agent CLI) — two sibling AI coding CLIs that
Claude Code can hand work off to for a second opinion or parallel work,
tag-team style.

See [SETUP.md](./SETUP.md) for full installation steps, usage examples, and
security notes (read that section before sharing tokens or config files —
each user authenticates with their own account).

Includes the actual `.claude/skills/zcode/` and `.claude/skills/agy/`
Claude Code Skill files, so you can drop them into any project (or your
user-level `~/.claude/skills/`) and use `/zcode` / `/agy` directly inside
Claude Code.
