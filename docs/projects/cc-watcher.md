# cc-watcher.nvim

Neovim plugin that monitors [Claude Code](https://claude.ai/claude-code) changes in real time. See what Claude is editing in a sidebar, view inline diffs, and navigate between hunks — all without leaving your editor.

Designed for a **tmux split workflow**: Claude Code on the left, Neovim on the right.

## Features

- **Sidebar** — tree-style file list with directory grouping, modification times, and +N/-M diff stats
- **Inline diff** — colored highlights showing exactly what changed
- **Hunk navigation** — jump between changes, revert individual hunks
- **Commit history** — browse past commits where Claude edited files
- **Session awareness** — reads all JSONL session logs for multi-session support
- **MCP bridge** — WebSocket server for direct Claude Code communication
- **14 integrations** — snacks, fzf-lua, trouble, diffview, conform, neotest, gitsigns, neo-tree, edgy, fidget, overseer, flash, mini.diff, notifier

## Screenshots

### Sidebar
![Sidebar](https://raw.githubusercontent.com/elmomk/cc-watcher.nvim/main/assets/sidebar.png)

### Inline Diff
![Inline Diff](https://raw.githubusercontent.com/elmomk/cc-watcher.nvim/main/assets/inline-diff.png)

### Snacks Picker
![Snacks Picker](https://raw.githubusercontent.com/elmomk/cc-watcher.nvim/main/assets/snacks-picker.png)

### Diffview
![Diffview](https://raw.githubusercontent.com/elmomk/cc-watcher.nvim/main/assets/diffview.png)

## Quick Start

```lua
{
    "elmomk/cc-watcher.nvim",
    event = { "BufReadPost", "BufNewFile" },
    cmd = { "ClaudeSidebar", "ClaudeDiff", "ClaudeSnacks" },
    keys = {
        { "<leader>cs", desc = "Claude - toggle sidebar" },
        { "<leader>cd", desc = "Claude - toggle inline diff" },
    },
    opts = {},
}
```

<div class="project-banner claude" style="margin-top: 3rem;">

<a href="https://github.com/elmomk/cc-watcher.nvim" class="md-button">View on GitHub</a>

</div>
