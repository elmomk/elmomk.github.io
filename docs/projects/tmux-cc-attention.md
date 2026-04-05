# :material-bell: tmux_cc_attention

!!! quote "Visual indicators for Claude Code session state — know when Claude needs you"

## Indicators

| State | Symbol | Color | Meaning |
|-------|:------:|:-----:|---------|
| Active | `*` | :material-circle:{ style="color: #10b981" } Green | Claude Code is working |
| Attention | `!` | :material-circle:{ style="color: #ef4444" } Red | Claude Code needs your input |
| Stopped | `-` | :material-circle:{ style="color: #3b82f6" } Blue | Claude Code has stopped |

## Features

- :material-pin: **Persistent state** — colors stay until actual state changes
- :material-monitor-multiple: **Cross-session** — status bar shows counts from other sessions (`!3 *2 -1`)
- :material-dock-window: **Session dashboard** — `prefix + G` popup to see all Claude windows (requires fzf)
- :material-check-circle: **Done notification** — inline indicator when Claude finishes
- :material-palette: **Themes** — Kanagawa Dragon, Catppuccin Mocha, Tokyo Night, Dracula

## Installation

!!! example "TPM (tmux Plugin Manager)"

    ```tmux
    set -g @claude-theme 'catppuccin-mocha'
    set -g @claude-popup-key 'G'
    set -g @claude-done-popup 'on'
    set -g @plugin 'elmomk/tmux_cc_attention'
    ```

<div class="project-banner claude" style="margin-top: 3rem;">

<a href="https://github.com/elmomk/tmux_cc_attention" class="md-button">View on GitHub</a>

</div>
