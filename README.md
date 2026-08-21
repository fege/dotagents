# dotagents

Personal TDD loop for [Claude Code](https://code.claude.com/docs/en/skills) and [Cursor](https://cursor.com/docs/skills): plan with the user, write failing tests, implement, then review. The planner never writes production code. Workers are one-shot; they finish the job or return a status. They do not interview you from inside a dead subprocess.

Worker **instructions** live once under `skills/`. Claude Code loads those files as skills. Cursor loads thin wrappers under `cursor/` that pin Grok/Composer and then read the same `SKILL.md` bodies.

```
you  -->  /planner  -->  agree on PLAN.md
                            |
                            v
                         tester
                            |
                            v
                  impl-low / impl-med / impl-high
                            |
                            v
                    you run the tests
                            |
                            v
                      reviewer-fede
```

You can also call `/reviewer-fede` directly. A one-skill request (`review this PR`, `write tests for X`) skips the rest of the loop.

## What each piece does

| Piece | Role | Claude Code | Cursor |
| --- | --- | --- | --- |
| Planner | Orchestrator. Agrees a plan, writes `.claude/PLAN.md`, delegates. Does not implement. | [`prompts/planner.md`](prompts/planner.md) — Sonnet 5, medium, `/planner` command | [`cursor/skills/planner`](cursor/skills/planner/SKILL.md) — inherit chat model, `/planner` skill, Task tool |
| Tester | Failing tests first. Missing production code is expected. | [`skills/tester`](skills/tester/SKILL.md) — Sonnet 5, medium, fork | [`cursor/agents/tester.md`](cursor/agents/tester.md) — Grok 4.6 medium |
| impl-low | Trivial one-file / mechanical changes. | Haiku 4.5, low, fork | Composer 2.5 |
| impl-med | Standard multi-file features. | Sonnet 5, medium, fork | Grok 4.6 medium |
| impl-high | Cross-cutting or high-risk work. | Sonnet 5, high, fork | Grok 4.6 high |
| reviewer-fede | Independent review vs ticket and diff. Read-only. | Opus 4.8, medium, 1M, inline | Grok 4.6 xhigh, readonly subagent |

Canonical worker text: [`skills/*/SKILL.md`](skills/). Cursor agents do not copy it.

Workers return one of:

- `STATUS: DONE` — planner continues
- `STATUS: BLOCKED` — planner asks you if needed, then **re-invokes** the same worker with the missing fact. Do not answer a completed worker in chat.
- `STATUS: ESCALATE` — planner calls the named higher impl tier

`tester` / `impl-*` cannot ask multiple-choice questions. If `reviewer-fede` cannot identify the diff or ticket, it returns `STATUS: BLOCKED` instead of guessing.

## Install

Clone this repo, then symlink. Paths assume `$DOTAGENTS="$HOME/Code/dotagents"`.

### Claude Code

```bash
git clone git@github.com:fege/dotagents.git "$HOME/Code/dotagents"
DOTAGENTS="$HOME/Code/dotagents"

mkdir -p "$HOME/.claude/skills" "$HOME/.claude/commands"

ln -sfn "$DOTAGENTS/skills/tester"          "$HOME/.claude/skills/tester"
ln -sfn "$DOTAGENTS/skills/impl-low"        "$HOME/.claude/skills/impl-low"
ln -sfn "$DOTAGENTS/skills/impl-med"        "$HOME/.claude/skills/impl-med"
ln -sfn "$DOTAGENTS/skills/impl-high"       "$HOME/.claude/skills/impl-high"
ln -sfn "$DOTAGENTS/skills/reviewer-fede"   "$HOME/.claude/skills/reviewer-fede"
ln -sfn "$DOTAGENTS/prompts/planner.md"     "$HOME/.claude/commands/planner.md"
```

### Cursor

```bash
DOTAGENTS="$HOME/Code/dotagents"
mkdir -p "$HOME/.cursor/skills" "$HOME/.cursor/agents"

ln -sfn "$DOTAGENTS/cursor/skills/planner"      "$HOME/.cursor/skills/planner"
ln -sfn "$DOTAGENTS/cursor/agents/tester.md"    "$HOME/.cursor/agents/tester.md"
ln -sfn "$DOTAGENTS/cursor/agents/impl-low.md"  "$HOME/.cursor/agents/impl-low.md"
ln -sfn "$DOTAGENTS/cursor/agents/impl-med.md"  "$HOME/.cursor/agents/impl-med.md"
ln -sfn "$DOTAGENTS/cursor/agents/impl-high.md" "$HOME/.cursor/agents/impl-high.md"
ln -sfn "$DOTAGENTS/cursor/agents/reviewer-fede.md" "$HOME/.cursor/agents/reviewer-fede.md"
```

Use Grok 4.6 as the chat model when you run `/planner`. Do not mix Grok 4.5 and 4.6 effort variants in the same session (Composer 2.5 for `impl-low` is a different family and is fine). Cursor also loads `~/.claude/skills/` for compatibility; `/planner` must dispatch via **Task** subagents, not those Claude skill copies.

## How to use

In a project repo, start Claude Code or Cursor Agent and run `/planner`, then describe the work.

Typical loop:

1. Planner investigates and proposes an approach. It does not write `.claude/PLAN.md` until you agree.
2. After you agree, it delegates to `tester`. Tests go in first, even if the implementation does not exist yet.
3. On `STATUS: DONE`, it delegates to `impl-low`, `impl-med`, or `impl-high`. When unsure, it picks the lower tier.
4. You run the test command tester returned (planner will ask first). Do not start review until tests are green.
5. Planner invokes `reviewer-fede` against the ticket and the diff. Verdict is `BLOCK` or `SHIP-WITH-NITS`.

Skip the full loop when the request is already one worker's job: `/planner review this PR` should brief `reviewer-fede` and stop.

Do not invoke `tester` or `impl-*` from the `/` menu in Claude Code (`user-invocable: false`). In Cursor, let `/planner` launch them as subagents.
