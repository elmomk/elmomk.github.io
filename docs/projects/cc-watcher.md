# :material-eye: cc-watcher.nvim

!!! quote "Real-time file change monitor for Claude Code — sidebar, inline diffs, and 14 integrations"

    Designed for a **tmux split workflow**: Claude Code on the left, Neovim on the right.

## Features

<div class="grid cards" markdown>

-   :material-file-tree:{ .lg .middle } **Sidebar**

    ---

    Tree-style file list with directory grouping, modification times, and +N/-M diff stats.

-   :material-format-color-highlight:{ .lg .middle } **Inline Diff**

    ---

    Colored highlights showing exactly what changed — added, modified, deleted lines.

-   :material-swap-vertical:{ .lg .middle } **Hunk Navigation**

    ---

    `]c` / `[c` to jump between changes, `cr` to revert individual hunks.

-   :material-history:{ .lg .middle } **Commit History**

    ---

    Browse past commits where Claude edited files, view commit diffs in the sidebar.

-   :material-connection:{ .lg .middle } **MCP Bridge**

    ---

    WebSocket server (RFC 6455) for direct Claude Code communication.

-   :material-puzzle:{ .lg .middle } **14 Integrations**

    ---

    snacks, fzf-lua, trouble, diffview, conform, neotest, gitsigns, neo-tree, edgy, fidget, overseer, flash, mini.diff, notifier.

</div>

## Screenshots

=== "Sidebar"

    ![Sidebar](/assets/screenshots/cc-watcher/sidebar.png){ width="450" }

    *Tree-style file list with directory grouping and live stats*

=== "Inline Diff"

    ![Inline Diff](/assets/screenshots/cc-watcher/inline-diff.png){ width="450" }

    *Colored inline highlights showing what Claude changed*

=== "Snacks Picker"

    ![Snacks Picker](/assets/screenshots/cc-watcher/snacks-picker.png){ width="450" }

    *Fuzzy-find changed files with colored diff preview*

=== "Hunks"

    ![Snacks Hunks](/assets/screenshots/cc-watcher/snacks-hunks.png){ width="450" }

    *Browse individual hunks across all changed files*

=== "Trouble"

    ![Trouble](/assets/screenshots/cc-watcher/trouble.png){ width="450" }

    *Diagnostic-like list of all Claude changes*

=== "Diffview"

    ![Diffview](/assets/screenshots/cc-watcher/diffview.png){ width="450" }

    *Side-by-side diff: pre-Claude snapshot vs current file*

## Quick Start

!!! example "lazy.nvim"

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
