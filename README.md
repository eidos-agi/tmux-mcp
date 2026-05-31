# emux

> **Eidos mux.** Pick up where you left off in tmux. A TUI session picker for humans + an MCP server for agents — same registry, same sessions, same operating model.

## What it does

Two front-ends over one shared registry of named tmux sessions:

```
emux              → TUI picker. Lists registered + live sessions.
                    Pick one → tmux attach. Stale entries flagged.

emux mcp          → MCP server. Six tools for agents to drive
                    sessions: list, register, send, capture, run.

emux ls           → Print registered + live sessions (non-interactive,
                    CI-friendly).
emux watch        → Watch many registered/live sessions in one terminal.
emux register     → Register a session under a friendly name.
emux unregister   → Drop a registered name. Doesn't touch tmux.
```

The registry persists at `~/.config/emux/registry.json` (override via `$EMUX_REGISTRY`).

## Why it exists

Two motivating problems, one tool:

**For humans:** "Which tmux session was I working in?" After ten sessions accumulate, remembering which one had the long-running build, which one had the Claude Code chat with useful context, which one was a throwaway — that's the friction. emux's TUI shows the registered names with descriptions ("production claude session", "test-shell", "long backfill") and stale flags (sessions you registered but tmux has since reaped). Pick one, you're attached. No remembering tmux session ids.

**For agents:** When an agent in one Claude Code session needs to inspect, prompt, or steer a session running in another tmux pane — for handoff, debate, monitoring, or autonomous round-trip testing of marketplace installs — it needs structured access to send keys and read the result. emux's MCP server gives that without the agent owning session lifecycle.

The registry is the same surface for both. Register once interactively, drive forever from agents. Or vice versa.

## Install

Until the first PyPI release, run directly from Git or from a local checkout:

```bash
uvx --from git+https://github.com/eidos-agi/emux.git emux       # TUI picker
uvx --from git+https://github.com/eidos-agi/emux.git emux mcp   # MCP server
```

In a Claude Code marketplace plugin, the `.mcp.json` looks like:

```json
{"emux": {"command": "uv", "args": ["run", "--directory", "${CLAUDE_PLUGIN_ROOT}", "emux", "mcp"]}}
```

Local development:

```bash
git clone https://github.com/eidos-agi/emux
cd emux
uv sync
uv pip install -e ".[dev]"
uv run pytest
uv run ruff check .
```

## TUI picker

Running `emux` with no arguments opens a Textual picker with a filter box,
number-key shortcuts, grouped session lists, and a live preview pane:

```
Registered (live)
   1  ●  claude-prod  → main

Registered (stale)
   2  ●  long-build  → backfill

Unregistered live tmux
   3  ○  experiments  unregistered

Actions
   4  ⊕  (register new)
```

- **Registered + live** entries attach immediately on selection (`tmux attach -t <session>`).
- **Stale** registered entries warn that the underlying tmux session is gone; they do not attach.
- **Live but unregistered** entries attach on Enter and can be registered with `r`.
- **(register new)** prompts for `name`, `session id`, optional `description`, and tags, then optionally attaches.

The picker is a terminal UI, not a terminal owner. Sessions are registered with
Emux for discovery and attached via Emux when selected. tmux still owns the
session lifecycle.

## MCP server

Six tools, exposed via `emux mcp`:

| Tool | What it does |
|---|---|
| `tmux_sessions()` | List live tmux sessions + registry (with stale flag) |
| `tmux_register(name, session, description?, tags?)` | Save friendly-name → session mapping with metadata |
| `tmux_unregister(name)` | Remove from registry; doesn't touch tmux |
| `tmux_send(target, keys, enter, by_registry_name)` | Send keystrokes |
| `tmux_capture(target, lines, by_registry_name)` | Read pane + scrollback |
| `tmux_run(target, command, wait_seconds, ...)` | Convenience: send + sleep + capture |

Example: agent drives a registered session.

```python
await tmux_register(
    name="claude-prod",
    session="main",
    description="production claude session",
    tags=["prod", "claude"],
)

result = await tmux_run(
    target="claude-prod",
    command="claude plugins marketplace update eidos-marketplace",
    wait_seconds=3,
    by_registry_name=True,
)
print(result["content"])  # tmux pane contents after the command
```

## Design principles

- **Existing sessions only.** Never spawns, never kills tmux sessions. Lifecycle is the user's. emux just observes and drives.
- **Registry is metadata only.** Live state always comes from `tmux list-sessions`. Stale entries are flagged, not auto-deleted — the user decides.
- **One registry for both surfaces.** TUI and MCP read and write the same JSON. Register interactively, drive from an agent. Or the reverse.
- **Textual TUI.** The picker uses Textual for filtering, preview, keyboard shortcuts, and grouped session state.
- **No magic, no recursion guards.** Sending `claude` keystrokes into a session that's already running emux's MCP gives you the recursion you asked for. Be deliberate.

## Storage

Registry JSON at `~/.config/emux/registry.json` (override via `$EMUX_REGISTRY`). Format:

```json
{
  "claude-prod": {
    "session": "main",
    "description": "production claude session",
    "tags": ["prod", "claude"],
    "registered_at": 1777400000
  }
}
```

For backwards compatibility with the prior name (`tmux-mcp`), `$TMUX_MCP_REGISTRY` is also honored if `$EMUX_REGISTRY` is unset.

## What it does NOT do

- **Doesn't spawn tmux sessions.** Use `tmux new-session` yourself; emux is read/drive only.
- **Doesn't bypass auth or approvals.** If the controlled session asks Claude Code for login, MFA, approval, or a human decision, Emux only sees and sends terminal text.
- **Doesn't strip ANSI.** Capture content includes raw bytes from tmux. Strip with `re.sub(r'\x1b\[[0-9;]*[a-zA-Z]', '', text)` if you need clean output.
- **Doesn't proxy MCP from inside tmux.** If the tmux session is running its own MCP server, emux only sees the stdin/stdout text — not the structured MCP messages.
- **Doesn't long-poll.** `tmux_run`'s `wait_seconds` is a fixed sleep. For long commands, prefer `tmux_send` + polling `tmux_capture` until you see the prompt return.

## Claude Code in tmux

Emux can control Claude Code when Claude Code is already running inside tmux:

```bash
tmux new -s claude-code
claude
```

From another terminal or agent, register and drive that existing session:

```bash
emux register claude-code claude-code -d "Claude Code terminal" -t claude local
```

Agents can then use `tmux_run(..., by_registry_name=True)` or separate
send/capture calls against `claude-code`.

## Watching many sessions

Use `emux watch` to watch all registered sessions plus live unregistered tmux
sessions in one refreshing terminal dashboard:

```bash
emux watch
emux watch --filter claude
emux watch --registered-only
emux watch --once --lines 12
```

This is a watcher, not a supervisor. It repeatedly captures visible pane
content with `tmux capture-pane`; it does not send input, create sessions, or
decide whether a Claude Code session is blocked.

## License

MIT — see [LICENSE](LICENSE).
