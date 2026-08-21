---
name: srota-cli
description: Drive Srota panes from a plain shell via the srota-cli binary — list/create/send/read/wait/close against the same srota-daemon socket the app and MCP server use, no MCP connection required. Also drives the iOS Simulator directly (screenshot/tap/swipe/home) via idb. Use this when srota-cli is on PATH but no mcp__srota__* tools are connected (a bare shell session, a script, or a CI-style context).
---

# srota-cli

`srota-cli` is a standalone binary that talks to the same srota-daemon Unix socket as the Srota app and the `srota` MCP server, but from a plain shell — no MCP connection needed. It's the CLI counterpart to the [[srota-mcp-orchestration]] skill.

**This skill applies when:** `srota-cli` is on PATH and no `mcp__srota__*` tools are connected — a bare terminal, a shell script, a cron/CI job, or any agent session without the MCP server wired up. If `mcp__srota__*` tools ARE available, use those instead (richer: fan-out budget, structured reports, session notes) — this binary has no equivalent for those.

Every subcommand except the simulator ones talks to `~/.srota/daemon.sock` (override with `SROTA_SOCKET_PATH`, or `SROTA_DIR` to change the directory). If no daemon is running, the command fails immediately with a message telling you to start the app or run `srota-daemon` directly.

## Simulator commands bypass the daemon

`screenshot`, `tap`, `swipe`, and `home` don't go through the daemon at all — they talk directly to the `idb_companion` process on `127.0.0.1:10882` that the Srota app's Simulator panel already has open (same connection `IdbClient` in the main app target uses). This means:

- The Simulator panel must already be open in the Srota app, or every one of these calls fails with a connection error.
- Coordinates for `tap`/`swipe` are device points (e.g. iPhone logical point space), not pixels — a screenshot's pixel dimensions are typically 2x or 3x the point coordinates you pass here.
- There's no single "click" HID event, only discrete press/release — `tap` and `home` each send a down/up pair; `swipe` is one `HIDSwipe` from start to end (idb has no "move while held" primitive for a multi-point drag).

## Commands

- `srota-cli list` — list all live panes as the daemon's raw JSON.
- `srota-cli create --preset <name|uuid> [--cwd <dir>] [--stable-id <id>] [--prompt <text>] [--rows <n>] [--cols <n>]` — create a pane and launch a Terminal Preset into it. `--cwd` defaults to your current directory; `--stable-id` defaults to a fresh UUID if omitted; `--prompt` is sent as the pane's first message.
- `srota-cli send <pane-id> <text> [--force] [--guard]` (alias `input`) — type text into a pane, as if the user typed it. See "Mid-turn guard" below.
- `srota-cli read <pane-id> [--replay-bytes <n>] [--readable]` (alias `attach`) — stream a pane's output until it closes. `--readable` strips ANSI escapes and collapses `\r`-redraws into flat text; without it you get raw JSON frames (`ring_buffer`/`live`/`dead`) on stdout, one per line.
- `srota-cli wait <pane-id> [--until <s1,s2,...>] [--timeout <ms>] [--expect-kind <kind>]` — block server-side until the pane's agent reaches one of the target statuses (e.g. `idle`, `blocked`, `done`).
- `srota-cli close <pane-id>` — close a pane.
- `srota-cli screenshot [--out <path>] [--max-width <px>]` — save a PNG of the iOS Simulator (default: `screenshot.png`, native retina resolution). Pass `--max-width` to downscale (aspect-preserved) — the native resolution (e.g. 1206x2622) is more pixels than a vision-model agent needs to read the screen, and the larger image costs more tokens/latency per screenshot->view->act round trip. `--max-width 400` is legible for typical iPhone UI at ~1/4 the file size.
- `srota-cli tap --x <x> --y <y> [--out <path>] [--max-width <px>]` — send a tap at a point (device points, not pixels).
- `srota-cli swipe --x1 <x> --y1 <y> --x2 <x> --y2 <y> [--out <path>] [--max-width <px>]` — send a swipe/drag between two points.
- `srota-cli home [--out <path>] [--max-width <px>]` — press the hardware Home button.
- `srota-cli --help` / `-h` — print the command surface.

Every command prints the daemon's raw JSON response to stdout (or, for `read --readable`, cleaned text instead). `screenshot` prints the saved file path instead; `tap`/`swipe`/`home` produce no stdout on success — unless `--out` is passed (see below), in which case they print the screenshot path like `screenshot` does.

### Combining an action with a screenshot

`tap`/`swipe`/`home` accept the same `--out`/`--max-width` flags as `screenshot`: pass `--out <path>` and the CLI performs the action, then immediately captures the resulting frame — one call and one round trip instead of the action followed by a separate `screenshot` call. This matters for an agent driving the simulator in a loop (tap → look → tap → look → ...): each saved round trip is a full extra model turn. Prefer `srota-cli tap --x 200 --y 400 --out /tmp/after.png --max-width 400` over two separate calls.

## Exit codes (for scripting)

- `0` — request succeeded.
- `1` — the daemon received the request but it failed (e.g. `error`, `agent_wait_timed_out`, `agent_wait_occupant_changed`, `agent_wait_kind_mismatch` responses), or the socket connection itself failed after connecting.
- `2` — usage error (missing/bad args, no daemon socket reachable, unknown command).

Don't parse stdout to detect failure in a script — check `$?`. A daemon error response (still valid JSON, e.g. `{"type":"error",...}`) exits `1`; only genuine usage mistakes exit `2`.

## Mid-turn guard on `send`

`send`/`input` mirrors the MCP server's `send_pane_input` guard: by default it does NOT check the target pane's agent status before typing — pass `--guard` to opt into the check (refuses if the pane is currently `working`, atomically checked server-side so there's no race between checking and typing), and `--force` to bypass the guard and type anyway even while `working`. Without either flag, the daemon just writes the input unconditionally. Use `--guard` when you don't want to interrupt a busy agent; use `--force` when you deliberately want to interrupt one.

`send` on a successful write with no refusal has nothing to report back — the wire protocol is fire-and-forget in that case, so the CLI prints `{"type":"sent"}` and exits `0` after a short grace period with no reply. A `--guard` refusal (or any other daemon error) comes back as a real JSON error response and exits `1`.

## `wait` blocks, it doesn't poll

`wait` is a single blocking request — the daemon holds the connection open server-side until the pane's agent reaches one of the `--until` statuses (or `--timeout` elapses). Don't wrap it in a polling loop; call it once and let it block. A timeout is a normal outcome (`agent_wait_timed_out`), not a crash — check the exit code (`1`) or response `type` to distinguish it from a resolved wait.

## `read --readable`

Plain `read`/`attach` streams raw terminal bytes wrapped in JSON frames — ANSI escapes, cursor movement, and carriage-return redraws all intact, same as what a real terminal would render. `--readable` strips ANSI sequences and collapses `\r`-redraws (keeping only what follows the last `\r` on each line) so scripts get flat, greppable text instead. Internally this is a one-shot snapshot (mirroring the MCP tool's `read_pane_output`): it prints the accumulated replay buffer once the daemon signals `ring_buffer_done` and stops, rather than continuing to block for further live output.
