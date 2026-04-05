# tmux_cc_attention

A tmux plugin that provides visual indicators for Claude Code session state.

## Indicators

| State | Symbol | Color | Meaning |
|-------|--------|-------|---------|
| Active | `*` | Green | Claude Code is working |
| Attention | `!` | Red | Claude Code needs your input |
| Stopped | `-` | Blue | Claude Code has stopped |

## Features

- Persistent state — colors stay until actual state changes
- Cross-session awareness — status bar shows counts from other sessions (`!3 *2 -1`)
- Session dashboard — `prefix + G` popup to see all Claude windows (requires fzf)
- Done notification — inline indicator when Claude finishes
- Themes — Kanagawa Dragon, Catppuccin Mocha, Tokyo Night, Dracula

## Installation (TPM)

```tmux
set -g @claude-theme 'catppuccin-mocha'
set -g @claude-popup-key 'G'
set -g @claude-done-popup 'on'
set -g @plugin 'elmomk/tmux_cc_attention'
```

<div class="project-banner claude" style="margin-top: 3rem;">

<a href="https://github.com/elmomk/tmux_cc_attention" class="md-button">View on GitHub</a>

</div>
