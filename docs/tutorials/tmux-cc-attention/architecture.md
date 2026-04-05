# Architecture

## Data Flow

```
                    ┌─────────────┐
                    │ Claude Code  │
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
     PreToolUse      Notification        Stop
           │               │               │
           ▼               ▼               ▼
       active.sh      notify.sh       stopped.sh
           │               │               │
           ▼               ▼               ▼
     fg = green       fg = red        fg = dark
     @claude-active   @claude-attention @claude-stopped
           │               │               │
           └───────────────┼───────────────┘
                           │
                    User switches to          User presses
                    the window (hook)         popup key
                           │                      │
                           ▼                      ▼
                       clear.sh              popup.sh
                           │                      │
                           ▼                      ▼
                  Unset format override    fzf dashboard
                  Unset all markers        switch-client
```

## State Machine

```
              active.sh                notify.sh
    ┌─────────────────────────┐  ┌──────────────────┐
    │                         ▼  │                   ▼
  NORMAL ─── active.sh ─── ACTIVE ── notify.sh ── ATTENTION
    ▲                         │                      │  │
    │         stopped.sh      │       active.sh      │  │
    │    ┌────────────────────┘      (user replied)  │  │
    │    ▼                              ┌────────────┘  │
  STOPPED ◄──── stopped.sh ────────────┘                │
    ▲                                                   │
    └──────────── clear.sh (from any state) ────────────┘
```

Priority: **ATTENTION > ACTIVE > STOPPED > NORMAL**

- `notify.sh` always wins — sets `@claude-attention`, clears `@claude-active` and `@claude-stopped`
- `active.sh` clears `@claude-attention` and `@claude-stopped`, short-circuits if already active
- `stopped.sh` clears `@claude-active` and `@claude-attention`
- `clear.sh` clears all markers and reverts to global format

A post-write race guard in `active.sh` re-checks `@claude-attention` after writing — if `notify.sh` set it concurrently, the attention format is restored.

## Session Dashboard Popup

`popup.sh` runs inside `tmux display-popup -E` when the user presses the configured key (`@claude-popup-key`). It:

1. Calls `tmux list-panes -a` to find all windows with a Claude process
2. Calls `tmux list-windows -a` to read state markers and window names
3. Formats each Claude window with ANSI-colored state icons
4. Pipes to `fzf --ansi` with a live `capture-pane` preview
5. On selection: `tmux switch-client -t session:window`

The keybinding is registered in `claude-attention.tmux` only if `@claude-popup-key` is set (opt-in).

## Done Notification

When `@claude-done-popup` is `on`, `stopped.sh` detects active→stopped transitions and sets `@claude-done-msg` (a global tmux option) with a colored inline message. This is rendered in `status-right` via `#{@claude-done-msg}`, appearing alongside cross-session counts. A background process clears the option after 5 seconds.

## Cross-Session Status

`status.sh` is called by tmux's `#()` in `status-right`. Three calls with different color wrappers:

```
#[fg=red]#(status.sh --attention)#[fg=green]#(status.sh --active)#[fg=dark]#(status.sh --stopped)
```

Each call runs `tmux list-windows -a`, filters out windows in the current session, and outputs `session:win` pairs for windows with the relevant marker.

## Hook Registration

Hooks use explicit array indices (`[100]`) instead of `-ga` (global append):

```bash
tmux set-hook -g 'session-window-changed[100]' "run-shell '.../clear.sh'"
tmux set-hook -g 'client-session-changed[100]' "run-shell '.../clear.sh'"
```

Reloading replaces the same index rather than appending duplicates. Old v1 hooks at indices 0-5 are cleaned up on load.

## State Storage

All state lives in **tmux window options** — no temp files:

| Option | Scope | Values | Purpose |
|--------|-------|--------|---------|
| `@claude-active` | per-window | `1` or unset | Claude is actively working |
| `@claude-attention` | per-window | `1` or unset | Claude needs user attention |
| `@claude-stopped` | per-window | `1` or unset | Claude has stopped |
| `window-status-format` | per-window override | format string | Visual color change |
| `@claude-popup-key` | global | key name or unset | Keybinding for session dashboard popup |
| `@claude-done-popup` | global | `on` or unset | Show message on active→stopped transition |

Clearing means unsetting the per-window override, which reverts to the global format.

## Themes

Four bundled themes in `themes/`. Each sets:
- Status bar structure and colors
- Window label formats (purple for selected, theme colors for normal)
- Plugin defaults (`@claude-window-bg`, `@claude-active-color`, `@claude-attention-color`, `@claude-stopped-color`)

The selected window tab uses the theme's purple/accent — green is reserved for active Claude sessions to avoid confusion.

## Design Decisions

### Why tmux options as state store?
- Atomic: tmux handles concurrency
- No cleanup needed: options live with the window
- Queryable: `list-windows -F` can read custom options in bulk

### Why template format strings instead of sed?
With a custom theme we control the format and build it directly via `get_window_format()`. No regex fragility.

### Why indexed hooks instead of `-ga`?
`-ga` appends on every reload, causing duplicates. Indexed hooks (`[100]`) are idempotent.

### Why `PreToolUse` for active state?
It fires whenever Claude uses a tool, reliably indicating active work. The script short-circuits after the first call (`@claude-active` already set) so subsequent calls are cheap.
