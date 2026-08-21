# dotagents

Personal [Claude Code](https://code.claude.com/docs/en/skills) skills for a TDD loop: plan with the user, write failing tests, implement, then review. The planner never writes production code. Workers are one-shot forks; they finish the job or return a status. They do not interview you from inside a dead subprocess.

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

| Piece | Role | How it runs |
| --- | --- | --- |
| [`prompts/planner.md`](prompts/planner.md) | Orchestrator. Investigates, gets your agreement, writes `.claude/PLAN.md`, then delegates. Does not implement. | `/planner` command (inline) |
| [`skills/tester`](skills/tester/SKILL.md) | Writes failing tests first. Missing production code is expected. | Fork, blocking. Planner-only. |
| [`skills/impl-low`](skills/impl-low/SKILL.md) | Trivial one-file / mechanical changes. Haiku. | Fork, blocking. Planner-only. |
| [`skills/impl-med`](skills/impl-med/SKILL.md) | Standard features and multi-file fixes. Sonnet. | Fork, blocking. Planner-only. |
| [`skills/impl-high`](skills/impl-high/SKILL.md) | Cross-cutting or high-risk work. Opus. | Fork, blocking. Planner-only. |
| [`skills/reviewer-fede`](skills/reviewer-fede/SKILL.md) | Independent review against the ticket and the actual diff. Read-only. | Inline on purpose, so you see the full review. Also `/reviewer-fede`. |

Forked workers return one of:

- `STATUS: DONE` — planner continues
- `STATUS: BLOCKED` — planner asks you if needed, then **re-invokes** the same skill with the missing fact. Do not answer a completed fork in chat.
- `STATUS: ESCALATE` — planner calls the named higher impl tier

`tester` / `impl-*` cannot ask multiple-choice questions. `reviewer-fede` is not forked; if it cannot identify the diff or ticket, it returns `STATUS: BLOCKED` instead of guessing.

## Install

Clone this repo, then symlink into Claude Code's personal skills and commands. Paths below assume the clone lives at `$DOTAGENTS`.

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

Claude Code follows these symlinks. Edits in this repo are live in the current session (`SKILL.md` text is watched; restart if you create a new top-level skills directory).

## How to use

In a project repo, start Claude Code and run `/planner`, then describe the work.

Typical loop:

1. Planner investigates and proposes an approach. It does not write `.claude/PLAN.md` until you agree.
2. After you agree, it delegates to `tester`. Tests go in first, even if the implementation does not exist yet.
3. On `STATUS: DONE`, it delegates to `impl-low`, `impl-med`, or `impl-high`. When unsure, it picks the lower tier.
4. You run the test command tester returned (planner will ask first). Do not start review until tests are green.
5. Planner invokes `reviewer-fede` against the ticket and the diff. Verdict is `BLOCK` or `SHIP-WITH-NITS`.

Skip the full loop when the request is already one skill's job: `/planner review this PR` should brief `reviewer-fede` and stop. You can also invoke `/reviewer-fede` yourself.

Do not invoke `tester` or `impl-*` from the `/` menu. They are `user-invocable: false` so only the planner (or Claude) should call them, with a complete argument string.
