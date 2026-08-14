---
name: srota-mcp-orchestration
description: Orchestrate sibling agent panes from inside a Srota terminal pane using Srota's MCP server — spawn helper agents, check their live status, read their output, and leave breadcrumb notes. Use this when running as an agent inside the Srota app and the task would benefit from delegating a sub-task to another agent pane, checking on work happening in a sibling pane, or coordinating multiple agents in the same workspace.
---

# Srota MCP Orchestration

Srota is a terminal/agent workspace app. When you're running as an agent inside one of its panes, an MCP server named `srota` may be connected, exposing tools to spawn other agent panes next to you, watch their status, and read their output — without the user manually managing multiple terminal windows.

This skill only applies when those tools are actually available (look for tool names prefixed `mcp__srota__` or similarly namespaced). If they aren't present, you're not running inside Srota — skip this skill.

## Tools

- `list_presets` — list Srota's Terminal Presets (Settings > Terminal), returning the exact names/UUIDs `spawn_pane`'s `preset` argument accepts. Call this before `spawn_pane` if you don't already know a valid preset name.
- `spawn_pane(direction, preset, task?, system_prompt?)` — split a new pane (`right`, `bottom`, or `newTab`) in your own workspace, running the given preset, optionally sending `task` (and `system_prompt`) as its first message. Blocks until the spawned agent actually reaches `idle`/`blocked`/`done` — or the wait times out (returns `status: "timed_out"` rather than failing) — then returns its `pane_id`. Never disturbs the user's current view. Refuses once this pane already has 16 live spawned descendants (transitive); see "Fan-out budget" below.
- `list_workspace_agents` — list the live status (`working` / `idle` / `blocked` / `done`) of every agent pane in your workspace, including ones `spawn_pane` created. Each entry also carries `spawned_by` (which pane spawned it, if any), `is_my_descendant` (whether this pane owns it, transitively), and `report` (that pane's latest `report_to_parent` call, as `{status, summary}`, if it sent one).
- `read_pane_output(pane_id, bytes?)` — read a pane's recent terminal scrollback (ANSI stripped). A snapshot, not a live tail — call it again to see new output.
- `send_pane_input(pane_id, text, press_enter?, force?)` — type raw text directly into another pane's terminal, as if the user typed it. This is real keystroke injection, not a message. Refuses by default when the target is `working`; see "Safety rules" below.
- `report_to_parent(summary, status?)` — report your result back to whoever spawned you via `spawn_pane`. The parent can't see your terminal output, so this is the only way it learns what you concluded. `status` is one of `completed`/`blocked`/`failed` (default `completed`). No-op, not an error, if you weren't spawned by another pane.
- `read_agent_reports(unread_only?)` — read reports left by agents this pane spawned. `unread_only` defaults to `true` (only reports since your last read); pass `false` to see the full history. Structured, survives the child's pane closing, and much cheaper than scraping scrollback via `read_pane_output`.
- `add_session_note(title, description?)` — log a short progress note to your own pane's timeline, visible live in the Srota app.
- `list_repos` / `add_repo(name, url?, default_branch?)` — list or register repos Srota knows about.

## Delegating work to a helper pane

1. `list_presets` to find a preset that matches the agent/environment you want (e.g. a specific CLI or working directory setup).
2. `spawn_pane` with `direction` and `preset`, passing `task` (and `system_prompt` if the helper needs role framing) as its first instruction if you already know exactly what it should do.
3. `spawn_pane` already blocked until the new pane reached `idle`/`blocked`/`done` before returning, so there's no need to immediately poll `list_workspace_agents` just to confirm it's alive. If it came back `status: "timed_out"`, that's your cue to check in.
4. Once the helper is working, prefer `read_agent_reports` over polling `read_pane_output` to collect what it found — it's structured, survives the child's pane closing, and much cheaper than scraping scrollback. Agent-preset helpers are launched with a standing instruction to call `report_to_parent` when they finish, so check back with `read_agent_reports` rather than watching their terminal. Fall back to `read_pane_output` only for raw detail a report didn't capture.

## Fan-out budget

Each pane may have at most 16 live spawned descendants at once — counted transitively across the whole subtree it spawned, not just direct children. `spawn_pane` refuses outright once you're at the cap; let some finish (their panes closing frees the slot automatically) before spawning more. This is an enforced backstop against a runaway spawn loop filling the user's machine with panes, not a limit meant to bite during ordinary use.

## Safety rules — read before calling `send_pane_input`

- **The mid-turn guard is server-enforced, not an honor rule.** `send_pane_input` refuses by default when the target pane's status is `working` — the daemon checks and writes as one atomic step, so there's no gap between checking status and typing. Wait for `idle`, or pass `force: true` if you genuinely mean to interrupt.
- **Status can still change out from under you.** The guard checks at write time, not when you last polled, so a pane that goes from `idle` back to `working` a moment before your call still gets the atomic check — but if you want to see *why* before interrupting, `read_pane_output` first to eyeball the current prompt state.
- **Stay workspace-scoped.** `spawn_pane` only ever creates panes next to you, in your own workspace — there's no tool here for targeting an arbitrary pane elsewhere in the app.

## Leaving breadcrumbs

Call `add_session_note` when your plan changes, you finish a meaningful chunk of work, or you're about to do something the user should see coming (a risky command, a big refactor, switching approach). This is an honor-system signal surfaced live in the Srota UI — there's no enforcement and no auto-surfacing, so it's only useful if you actually call it at the moments that matter. Don't call it for every small step; a note per meaningful checkpoint, not per tool call.
