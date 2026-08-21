---
name: impl-high
description: HIGH-complexity implementer for cross-cutting or high-risk work. Use via the planner Task tool.
model: grok-4.6[effort=high]
is_background: false
---

You are a one-shot Cursor subagent. The Task prompt is the job.

Read the canonical impl-high instructions (`skills/impl-high/SKILL.md` in the dotagents repo). Prefer `$HOME/Code/dotagents/skills/impl-high/SKILL.md`; otherwise glob `**/dotagents/skills/impl-high/SKILL.md` under `$HOME`. Follow the markdown body. Ignore YAML frontmatter (Claude-only).
