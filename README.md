# dotagents

Personal TDD loop for [Claude Code](https://code.claude.com/docs/en/skills), [Cursor](https://cursor.com/docs/skills), [Codex](https://developers.openai.com/codex/skills), and [OpenCode v2](https://opencode.ai/v2/docs/agents): plan with the user, write failing tests, implement, then review. The planner never writes production code. Workers are one-shot; they finish the job or return a status. They do not interview you from inside a dead subprocess.

Worker **instructions** live once under `skills/`. Claude Code loads those files as skills. Cursor, Codex, and OpenCode load thin wrappers (`cursor/`, `codex/`, `opencode/`) that pin models and then read the same `SKILL.md` bodies.

```
you  -->  /planner or $planner  -->  agree on plan + slices
                            |
                            v
              per slice:  tester  -->  impl-*  -->  you review  -->  tests
                            |
                            +-- more slices? ask, then loop
                            |
                            v
              reviewer-fede once (accumulated diff vs ticket)
```

You can also call `/reviewer-fede` directly. A one-skill request (`review this PR`, `write tests for X`) skips the rest of the loop.

## What each piece does

| Piece | Role | Claude Code | Cursor | Codex |
| --- | --- | --- | --- | --- |
| Planner | Orchestrator. Agrees a plan, writes `.plans/PLAN-YYYYMMDD-HHMMSS.md`, delegates. Does not implement. | [`prompts/planner.md`](prompts/planner.md) — Sonnet 5, medium, `/planner` command | [`cursor/skills/planner`](cursor/skills/planner/SKILL.md) — inherit chat model, `/planner` skill, Task tool | [`codex/skills/planner`](codex/skills/planner/SKILL.md) — `$planner` skill, spawn named agents |
| Tester | Failing tests first. Missing production code is expected. | [`skills/tester`](skills/tester/SKILL.md) — Sonnet 5, medium, fork | [`cursor/agents/tester.md`](cursor/agents/tester.md) — Grok 4.6 medium | [`codex/agents/tester.toml`](codex/agents/tester.toml) — GPT-6 Luna, max |
| impl-low | Trivial one-file / mechanical changes. | Haiku 4.5, low, fork | Composer 2.5 | GPT-6 Luna, high |
| impl-med | Standard multi-file features (one slice). | Sonnet 5, medium, fork | Grok 4.6 medium | GPT-6 Luna, max |
| impl-high | Cross-cutting or high-risk work. | Sonnet 5, high, fork | Grok 4.6 high | GPT-6 Sol, high |
| reviewer-fede | Independent review vs ticket and diff. Read-only. | Opus 4.8, medium, 1M, inline | Grok 4.6 xhigh, readonly subagent | GPT-6 Sol, medium, read-only sandbox |

Canonical worker text: [`skills/*/SKILL.md`](skills/). Cursor, Codex, and OpenCode adapters do not copy it.

Workers return one of:

- `STATUS: DONE` — planner continues
- `STATUS: BLOCKED` — planner asks you if needed, then **spawns a new** worker of the same role with the missing fact. Do not answer a completed worker in chat. Do not resume the old child.
- `STATUS: ESCALATE` — planner calls the named higher impl tier

`tester` / `impl-*` cannot ask multiple-choice questions. If `reviewer-fede` cannot identify the diff or ticket, it returns `STATUS: BLOCKED` instead of guessing.

## Install

Clone this repo, then symlink. Paths assume `$DOTAGENTS="$HOME/Code/dotagents"`.

### Claude Code

```bash
git clone git@github.com:fege/dotagents.git "$HOME/Code/dotagents"
DOTAGENTS="$HOME/Code/dotagents"

mkdir -p "$HOME/.claude/skills" "$HOME/.claude/commands" "$HOME/.claude/rules"

ln -sfn "$DOTAGENTS/skills/tester"          "$HOME/.claude/skills/tester"
ln -sfn "$DOTAGENTS/skills/impl-low"        "$HOME/.claude/skills/impl-low"
ln -sfn "$DOTAGENTS/skills/impl-med"        "$HOME/.claude/skills/impl-med"
ln -sfn "$DOTAGENTS/skills/impl-high"       "$HOME/.claude/skills/impl-high"
ln -sfn "$DOTAGENTS/skills/reviewer-fede"   "$HOME/.claude/skills/reviewer-fede"
ln -sfn "$DOTAGENTS/prompts/planner.md"     "$HOME/.claude/commands/planner.md"
ln -sfn "$DOTAGENTS/claude/rules/planner-tdd-dispatch.md" "$HOME/.claude/rules/planner-tdd-dispatch.md"
```

### Cursor

```bash
DOTAGENTS="$HOME/Code/dotagents"
mkdir -p "$HOME/.cursor/skills" "$HOME/.cursor/agents" "$HOME/.cursor/rules"

ln -sfn "$DOTAGENTS/cursor/skills/planner"      "$HOME/.cursor/skills/planner"
ln -sfn "$DOTAGENTS/cursor/agents/tester.md"    "$HOME/.cursor/agents/tester.md"
ln -sfn "$DOTAGENTS/cursor/agents/impl-low.md"  "$HOME/.cursor/agents/impl-low.md"
ln -sfn "$DOTAGENTS/cursor/agents/impl-med.md"  "$HOME/.cursor/agents/impl-med.md"
ln -sfn "$DOTAGENTS/cursor/agents/impl-high.md" "$HOME/.cursor/agents/impl-high.md"
ln -sfn "$DOTAGENTS/cursor/agents/reviewer-fede.md" "$HOME/.cursor/agents/reviewer-fede.md"
ln -sfn "$DOTAGENTS/cursor/rules/planner-tdd-dispatch.mdc" "$HOME/.cursor/rules/planner-tdd-dispatch.mdc"
```

Use Grok 4.6 as the chat model when you run `/planner`. Do not mix Grok 4.5 and 4.6 effort variants in the same session (Composer 2.5 for `impl-low` is a different family and is fine). Cursor also loads `~/.claude/skills/` for compatibility; `/planner` must dispatch via **Task** subagents, not those Claude skill copies.

### Codex

```bash
DOTAGENTS="$HOME/Code/dotagents"
mkdir -p "$HOME/.agents/skills" "$HOME/.codex/agents"

ln -sfn "$DOTAGENTS/codex/skills/planner" "$HOME/.agents/skills/planner"
# Role files must be regular files. Codex 0.149+ refuses symlink *.toml
# under ~/.codex/agents/ ("agent type is currently not available").
cp -f "$DOTAGENTS/codex/agents/tester.toml"        "$HOME/.codex/agents/tester.toml"
cp -f "$DOTAGENTS/codex/agents/impl-low.toml"      "$HOME/.codex/agents/impl-low.toml"
cp -f "$DOTAGENTS/codex/agents/impl-med.toml"      "$HOME/.codex/agents/impl-med.toml"
cp -f "$DOTAGENTS/codex/agents/impl-high.toml"     "$HOME/.codex/agents/impl-high.toml"
cp -f "$DOTAGENTS/codex/agents/reviewer-fede.toml" "$HOME/.codex/agents/reviewer-fede.toml"
```

Do **not** symlink `skills/tester` (etc.) into `~/.agents/skills`. Codex would load those as in-session skills and skip the one-shot spawn loop. `$planner` must spawn named agents under `~/.codex/agents/` with `agent_type` + `fork_turns = none`. Never `create_thread`. Never `followup_task` / resume a finished child — each TDD loop is a new `spawn_agent` with a new `task_name`. GPT-6 hides `agent_type` unless `~/.codex/config.toml` has:

```toml
# Do not set multi_agent_v2 = true. These keys only restore agent_type on spawn.
[features.multi_agent_v2]
hide_spawn_agent_metadata = false
tool_namespace = "agents"
```

The planner skill has `allow_implicit_invocation: false` — type `$planner`. Fully quit Codex Desktop after copying TOMLs or changing that block. Re-copy the TOMLs after you edit `codex/agents/` in this repo.

Verify spawns with `python3 $DOTAGENTS/codex/scripts/list_sessions.py`. That lists today's rollouts **and** parent threads they point at (the planner root often lives in an earlier day's folder). `kind=subagent` plus `agent_role` is a named custom agent. `kind=root` is the user/planner thread. `kind=thread` is a Desktop `create_thread` sibling (wrong model). `--no-parents` limits the list to that day's folder only. After a tester spawn, `python3 $DOTAGENTS/codex/scripts/check_spawn.py --role tester` must print `PASS`: it reads nick/role/model/effort from that rollout (`kind=subagent`, fields present). It does not parse agent TOMLs.

### OpenCode v2 (EnMaaS)

The native v2 adapters live in [`opencode/agents`](opencode/agents/) and
[`opencode/commands`](opencode/commands/). They use the `enmaas` provider already
configured in your OpenCode profile; they do not add providers or change the
default model. Validated against OpenCode 2.0.28.

| Agent | Model |
| --- | --- |
| planner | Inherits the selected chat model |
| tester | `enmaas/gpt-6-luna#xhigh` |
| impl-low | `enmaas/gpt-6-luna#high` |
| impl-med | `enmaas/gpt-6-luna#xhigh` |
| impl-high | `enmaas/gpt-6.1-sol#high` |
| reviewer-fede | `enmaas/gpt-6.1-sol#medium` |

`xhigh` is the highest effort variant exposed by this OpenCode provider, not a
literal `max` value copied from Codex. Change a wrapper's `model` to another
registered `enmaas/<model>#<variant>` to change that worker; changing the chat
model only changes the planner.

Install into the same config directory your CLI and desktop server already use:

```bash
DOTAGENTS="$HOME/Code/dotagents"
OPENCODE_CONFIG_DIR="$HOME/.config/opencode-enmaas/opencode"
export OPENCODE_CONFIG_DIR
mkdir -p "$OPENCODE_CONFIG_DIR/agents" "$OPENCODE_CONFIG_DIR/commands"

for role in planner tester impl-low impl-med impl-high reviewer-fede; do
  ln -sfn "$DOTAGENTS/opencode/agents/$role.md" "$OPENCODE_CONFIG_DIR/agents/$role.md"
done
ln -sfn "$DOTAGENTS/opencode/commands/planner.md" "$OPENCODE_CONFIG_DIR/commands/planner.md"
ln -sfn "$DOTAGENTS/opencode/commands/reviewer-fede.md" "$OPENCODE_CONFIG_DIR/commands/reviewer-fede.md"
```

Do not link the canonical Claude command or worker skills as OpenCode agents:
their frontmatter and dispatch dialect differ. The OpenCode wrappers read their
shared bodies and supply native `subagent` dispatch. Every call omits `sessionID`
and waits in the foreground, so retries and comment cycles create fresh workers.

In the CLI, start `opencode` from the target repo and type `/planner <request>`.
In desktop, open that repo on the existing EnMaaS server and use the same command.
For a standalone review use `/reviewer-fede <diff/branch/commit/PR>`, or
`/planner review <target>`. For noninteractive planner entry:

```bash
opencode run --agent planner 'Use the dotagents planner workflow to plan <request>.'
```

The planner inherits the selected model. Workers keep their pinned models.
Do not select an implementation worker as the main chat agent. Switch back to
`build` or start a normal new chat to leave the planner workflow.

Safety: native file edits are denied for the reviewer, and the planner can edit
only `.plans/*.md`. Workers cannot delegate or call the question tool. All these
agents require shell approval: approve only read-only investigation for workers.
This is not an OS-level read-only sandbox; never approve a shell command that
bypasses file restrictions. Tests, builds, formatters, and mutating git remain
human-gated. A legacy plan can be read as-is, but an amendment outside `.plans/`
needs a deliberate permission change, not a shell workaround.

For OpenCode post-implementation review, anchor the handoff to the parent user
message that dispatched the workers, not the newest/“Last turn” after later chat.
Track the parent session ID, dispatch-turn message ID, and returned tester and
implementation child session IDs (and actual native links, if provided) as
workflow state, using fresh child IDs/links for each retry/comment cycle; do not
edit a frozen plan to store them. When the IDs and a recorded snapshot are
available, the planner may inspect the accumulated parent snapshot diff for that
dispatch turn with
`GET /api/session/{parentID}/diff?from={dispatchMessageID}` read-only. Snapshot
capture is best-effort and location-dependent; an API diff does not prove that
child edits appear in a desktop parent-turn change view. Point to that native view
only if the client exposes that specific turn and its relevant content is
verified. Otherwise point to both child sessions and have the user inspect the
working-tree diff. Keep feedback in the parent chat: no automatic forwarding of
child inline comments is established. The planner may inspect accessible
submitted child messages/inbox read-only; if feedback is not available there,
ask the user to paste or forward it into the parent. Preserve the human
comment-versus-proceed gate.

The reviewer returns the full review for the planner to show before asking
which findings to address.

## How to use

In a project repo, start Claude Code, Cursor Agent, or OpenCode and run `/planner`, or start Codex and run `$planner`, then describe the work.

Typical loop:

1. Planner investigates and proposes an approach **and slices** (one reviewable behavior each). It does not write `.plans/PLAN-YYYYMMDD-HHMMSS.md` until you agree. There is no extra slicer agent.
2. For the **current slice**, it delegates to `tester`. Tests go in first, even if the implementation does not exist yet. Coverage is that slice only.
3. On `STATUS: DONE`, it delegates to `impl-low`, `impl-med`, or `impl-high` as named on that slice. When unsure, it picks the lower tier. No fourth impl role.
4. After impl, open the worker thread's native file-review UI (inline hunks / Edited files), then tell the planner to comment or proceed. Do not run tests or invoke `reviewer-fede` until you choose proceed.
5. You run the test command tester returned (planner will ask first). After this slice is green, the planner asks before starting the next slice. Do not invoke `reviewer-fede` between slices.
6. When the slices you want in this chat are done, planner invokes `reviewer-fede` once against the ticket and the **accumulated** diff. After the first review, you decide whether to run tester+impl at all. After those fixes are green, you decide whether to review again — not automatic. Verdict is `BLOCK` or `SHIP-WITH-NITS`.

Skip the full loop when the request is already one worker's job: `/planner review this PR` (or `$planner review this PR`) should brief `reviewer-fede` and stop.

Do not invoke `tester` or `impl-*` from the `/` menu in Claude Code (`user-invocable: false`). In Cursor and Codex, let the planner launch them as subagents.

## Plans

Plans live in the **project** repo as `.plans/PLAN-YYYYMMDD-HHMMSS.md` (UTC). Claude Code, Cursor, Codex, and OpenCode share that directory.

The plan file lists ordered slices (id, files, do-not-touch, impl tier, out of scope). One TDD cycle per slice. After you agree, that file is **frozen**. The planner must not rewrite it mid-loop (reslice, retarget files, mark slices done). A later edit needs an explicit yes to that amendment — "continue" / "proceed" / next slice is not enough.

A new chat does not resume an old plan. Leftover `PLAN*.md` files are not an entry signal. To continue previous work, invoke `/planner` or `$planner` and name the file:

```
/planner use .plans/PLAN-20260914-101800.md
```

That path is reused as-is until you approve a specific change. If you do not name a path, it agrees a new plan and writes a new timestamped file. Legacy `.claude/PLAN*.md` or `.dotagents/PLAN*.md` files can be named the same way.
