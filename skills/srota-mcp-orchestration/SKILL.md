---
name: srota-mcp-orchestration
description: Orchestrate sibling agent panes from inside a Srota terminal pane using Srota's MCP server — spawn helper agents, check their live status, read their output, and leave breadcrumb notes. Use this when running as an agent inside the Srota app and the task would benefit from delegating a sub-task to another agent pane, checking on work happening in a sibling pane, or coordinating multiple agents in the same workspace.
---

# Srota MCP Orchestration

Srota is a terminal/agent workspace app. When you're running as an agent inside one of its panes, an MCP server named `srota` may be connected, exposing tools to spawn other agent panes next to you, watch their status, and read their output — without the user manually managing multiple terminal windows.

This skill only applies when those tools are actually available (look for tool names prefixed `mcp__srota__` or similarly namespaced). If they aren't present, you're not running inside Srota — skip this skill.

## Tools

- `list_presets` — list Srota's Terminal Presets (Settings > Terminal), returning the exact names/UUIDs `spawn_pane`'s `preset` argument accepts. Call this before `spawn_pane` if you don't already know a valid preset name.
- `spawn_pane(direction, preset, task?)` — split a new pane (`right`, `bottom`, or `newTab`) in your own workspace, running the given preset, optionally sending `task` as its first message. Blocks until the pane exists and returns its `pane_id`. Never disturbs the user's current view.
- `list_workspace_agents` — list the live status (`working` / `idle` / `blocked` / `done`) of every agent pane in your workspace, including ones `spawn_pane` created.
- `read_pane_output(pane_id, bytes?)` — read a pane's recent terminal scrollback (ANSI stripped). A snapshot, not a live tail — call it again to see new output.
- `send_pane_input(pane_id, text, press_enter?)` — type raw text directly into another pane's terminal, as if the user typed it. This is real keystroke injection, not a message.
- `add_session_note(title, description?)` — log a short progress note to your own pane's timeline, visible live in the Srota app.
- `list_repos` / `add_repo(name, url?, default_branch?)` — list or register repos Srota knows about.

## Delegating work to a helper pane

1. `list_presets` to find a preset that matches the agent/environment you want (e.g. a specific CLI or working directory setup).
2. `spawn_pane` with `direction` and `preset`, passing `task` as the first instruction if you already know exactly what the helper should do.
3. Track it with the returned `pane_id`. Poll `list_workspace_agents` (or `read_pane_output` for detail) to see when it moves from `working` to `idle`/`done` — this is pull-based, there's no push notification, so don't loop tightly; check back periodically instead of spinning.
4. Read its output with `read_pane_output` once it looks idle or done, rather than interrupting it mid-task.

## Safety rules — read before calling `send_pane_input`

- **Never type into a pane that might be mid-turn.** `send_pane_input` is indistinguishable from the user typing — sending it to a busy agent corrupts its input stream. Only use it on a pane you just spawned (before it's started anything) or one `list_workspace_agents` confirms is `idle`.
- **Status is a snapshot, not a guarantee.** A pane reported `idle` a moment ago can start working again before your `send_pane_input` call lands. When in doubt, `read_pane_output` first to eyeball the current prompt state.
- **Stay workspace-scoped.** `spawn_pane` only ever creates panes next to you, in your own workspace — there's no tool here for targeting an arbitrary pane elsewhere in the app.

## Leaving breadcrumbs

Call `add_session_note` when your plan changes, you finish a meaningful chunk of work, or you're about to do something the user should see coming (a risky command, a big refactor, switching approach). This is an honor-system signal surfaced live in the Srota UI — there's no enforcement and no auto-surfacing, so it's only useful if you actually call it at the moments that matter. Don't call it for every small step; a note per meaningful checkpoint, not per tool call.
