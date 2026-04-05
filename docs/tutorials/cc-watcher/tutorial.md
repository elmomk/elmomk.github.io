# cc-watcher.nvim Tutorial

A hands-on guide to monitoring Claude Code changes in real time from inside Neovim.

---

## Table of Contents

1. [Getting Started](#1-getting-started)
2. [The Workflow](#2-the-workflow)
3. [Understanding the Sidebar](#3-understanding-the-sidebar)
4. [Working with Diffs](#4-working-with-diffs)
5. [Reviewing Commit History](#5-reviewing-commit-history)
6. [Git HEAD Diffing](#6-git-head-diffing-key-concept)
7. [Snacks Picker Integration](#7-snacks-picker-integration)
8. [LazyVim Dashboard Setup](#8-lazyvim-dashboard-setup)
9. [trouble.nvim Integration](#9-troublenvim-integration)
10. [Diffview Integration](#10-diffview-integration)
11. [Statusline Integration](#11-statusline-integration)
12. [Customization](#12-customization)
13. [Advanced Usage](#13-advanced-usage)
14. [Lazy Loading](#14-lazy-loading)
15. [Tips and Tricks](#15-tips-and-tricks)
16. [Troubleshooting](#16-troubleshooting)

---

## 1. Getting Started

### Prerequisites

- **Neovim >= 0.10** (required for `vim.uv`, `vim.diff`, and modern extmark features)
- **Claude Code** installed and functional (`claude` command available in your terminal)
- **git** (used as a fallback when no snapshot exists for a file)

### Installation with lazy.nvim

**Minimal setup** -- just the plugin with default settings:

```lua
{
    "elmomk/cc-watcher.nvim",
    config = function()
        require("cc-watcher").setup()
    end,
}
```

**Full setup** with lazy loading and all integrations enabled:

```lua
{
    "elmomk/cc-watcher.nvim",
    event = { "BufReadPost", "BufNewFile" },
    cmd = {
        "ClaudeSidebar", "ClaudeDiff",
        "ClaudeSnacks", "ClaudeTrouble", "ClaudeDiffview",
    },
    keys = {
        { "<leader>cs", desc = "Claude - toggle sidebar" },
        { "<leader>cd", desc = "Claude - toggle inline diff" },
    },
    config = function()
        require("cc-watcher").setup({
            sidebar_width = 100,
            keys = {
                toggle_sidebar = "<leader>cs",
                toggle_diff = "<leader>cd",
                snacks_files = "<leader>ct",
                snacks_hunks = "<leader>ch",
                trouble = "<leader>cx",
                diffview = "<leader>cv",
                flash = "<leader>cf",
            },
            integrations = {
                snacks = true,      -- requires snacks.nvim
                trouble = true,     -- requires trouble.nvim v3
                diffview = true,    -- no extra dependency (uses vim diff mode)
            },
        })
    end,
}
```

### Verify it works

1. Open a terminal and `cd` into any project directory.
2. Start Neovim in that directory: `nvim .`
3. Run `:ClaudeSidebar` -- you should see the sidebar open on the left with "Waiting for changes..."
4. In a separate terminal (same directory), start Claude Code and ask it to edit a file.
5. Watch the sidebar update in real time as Claude writes.

If the sidebar shows "session active" near the top, the plugin has found Claude Code's session. You are ready to go.

---

## 2. The Workflow

cc-watcher.nvim is designed for a **tmux split workflow**: Claude Code on one side, Neovim on the other.

### Step-by-step setup

**1. Create a tmux split.**

```bash
# Start tmux (if not already in a session)
tmux

# Split vertically: left pane for Claude, right for Neovim
tmux split-window -h
```

**2. Left pane: Claude Code.**

In the left pane, navigate to your project and start Claude:

```bash
cd ~/projects/my-app
claude
```

**3. Right pane: Neovim with cc-watcher.**

In the right pane, open Neovim in the same directory:

```bash
cd ~/projects/my-app
nvim .
```

Open the sidebar:

```
:ClaudeSidebar
```

**4. Ask Claude to edit files.**

In the left pane, give Claude a task:

```
> Add input validation to the create_user function in src/handlers.rs
```

**5. Watch the sidebar update.**

As Claude writes files, you will see entries appear in the sidebar in real time:

```
 Claude Code
  session active
────────────────────────────────────
  ● src/handlers.rs              +12 -3
  ● src/models.rs                 +5 -0

  2 files  +17 -3
  g? help
```

Files are shown as a flat list with full relative paths. The most recently edited file is highlighted with italic + warning color (`ClaudeFileLatest`). The `+N/-M` stats update live as the file content changes.

**6. Click into files to see inline diffs.**

Move your cursor to a file in the sidebar and press `<CR>` (or `d`). The file opens in the editor pane with full inline diff highlighting:

- Green background: lines Claude added
- Yellow background: lines Claude changed (with the old version shown above in red virtual text)
- Red virtual text: lines Claude deleted

**7. Navigate and review.**

Use `]c` to jump to the next hunk, `[c` to jump to the previous one. Each jump flashes the target line briefly so you can orient yourself.

If a hunk looks wrong, press `cr` while your cursor is inside it to **revert just that hunk** back to its original state. Other changes in the file are preserved.

---

## 3. Understanding the Sidebar

Open the sidebar with `<leader>cs` or `:ClaudeSidebar`. Here is what you see:

```
 Claude Code                        <- header
  session active                    <- Claude Code is running here
────────────────────────────────────  <- separator
  ● src/handlers.rs        +12 -3  <- live-detected, 12 added 3 deleted
  ● src/models.rs           +5 -0  <- live-detected, 5 added
  ○ tests/test_handlers.rs  +8 -2  <- session-detected

  3 files  +25 -5                   <- summary footer
  g? help                           <- press g? for help overlay
```

### Indicators

| Symbol | Meaning |
|--------|---------|
| `●` (yellow) | **Live change** -- detected by the file watcher as Claude wrote the file |
| `○` (blue) | **Session change** -- found in Claude Code's JSONL log (Write/Edit tool calls) |

Live changes are files you had open in Neovim when Claude edited them. The `fs_event` watcher caught the modification instantly.

Session changes are files Claude edited that you did **not** have open. The plugin found them by parsing Claude's JSONL session log at `~/.claude/projects/`.

### Flat file list

Files are shown as a flat list with full relative paths (e.g. `src/handlers.rs`), without directory grouping. The most recently edited file is highlighted with italic text and a warning color (`ClaudeFileLatest` highlight group), making it easy to spot what Claude just changed.

### Stats

The `+N/-M` next to each file shows how many lines were added and deleted compared to git HEAD (or snapshot fallback for untracked files). These stats are only computed for files that have a loaded buffer.

### Session status

- **"session active"** (green) -- Claude Code is running in this directory. The plugin found a valid session file at `~/.claude/sessions/` with a matching PID.
- **"no session"** (grey) -- No active Claude Code process found for this working directory.

### How detection works

**Live detection:** When you open a file in Neovim, cc-watcher registers a `libuv fs_event` watcher on it. If Claude (or anything else) modifies that file on disk, the watcher fires, the buffer auto-reloads (`checktime`), and the file appears in the sidebar with `●`.

**Session detection:** cc-watcher reads **all** Claude Code JSONL log files from `~/.claude/projects/` for the current project (multi-session). It does incremental tail-reads: only new bytes since the last parse are processed. It looks for `Write` and `Edit` tool calls and extracts the file paths. Files inside `.claude/` are filtered out, and only files inside the project directory with actual uncommitted diffs are shown. This catches files Claude edited even if you never opened them.

### Sidebar keybindings

| Key | Action |
|-----|--------|
| `<CR>` / `d` | Open file in editor pane with inline diff activated |
| `o` | Open file with diff |
| `H` | Toggle commit history mode |
| `]g` | Next commit (in history mode) |
| `[g` | Previous commit (in history mode) |
| `r` | Force refresh the file list |
| `q` | Close the sidebar |
| `g?` | Show help overlay |

The sidebar also refreshes automatically whenever a live change is detected or the JSONL log updates.

---

## 4. Working with Diffs

cc-watcher provides two levels of diff visualization:

### Sign column indicators (always on for changed files)

When Claude modifies a file you have open, lightweight sign-column markers appear automatically:

| Sign | Color | Meaning |
|------|-------|---------|
| `┃` | Green | Lines added |
| `┃` | Yellow | Lines changed |
| `▁` | Red | Lines deleted (marker at the deletion point) |

These are subtle indicators that show at a glance which regions changed. They appear as soon as a file is detected as changed, without needing to activate the full diff.

### Full inline diff

Toggle the full inline diff with `<leader>cd` or `:ClaudeDiff`. This is where things get detailed:

**Added lines** show with a green background and a green `┃` sign:

```
     let result = parse(data)?;
 ┃ + let validated = validate(&data);      <- green background
 ┃ + let normalized = normalize(validated); <- green background
     Ok(result)
```

**Changed lines** show the old version above (as red virtual text with `~` prefix) and the new version below with a yellow background:

```
   ~ let timeout = 30;                      <- red virtual text (old)
 ┃   let timeout = Duration::from_secs(60); <- yellow background (new)
```

**Deleted lines** appear as red virtual text with `-` prefix at the point they were removed:

```
     fn process(data: &str) {
   - let debug = true;                      <- red virtual text
   - println!("{:?}", data);                <- red virtual text
     let result = parse(data);
```

### Toggling

Running `<leader>cd` (or `:ClaudeDiff`) again on the same file **toggles the diff off**. The sign column indicators remain, but the full inline highlights and virtual text are removed.

### Navigating hunks

When the full inline diff is active, two buffer-local keybindings are set:

| Key | Action |
|-----|--------|
| `]c` | Jump to the next hunk |
| `[c` | Jump to the previous hunk |

Each jump centers the hunk on screen (`zz`) and flashes the line for 300ms using the `IncSearch` highlight, so you can immediately see where you landed.

At the last hunk, `]c` shows "No next hunk". At the first, `[c` shows "No previous hunk".

### Reverting hunks

Press `cr` with your cursor inside a hunk to **revert that specific hunk** to its pre-Claude state. The rest of the file is untouched.

For example, if Claude added 3 hunks to a file and you like 2 of them but not the third:

1. `]c` to navigate to the hunk you want to remove
2. `cr` to revert it
3. The diff re-renders automatically, now showing only the 2 remaining hunks

After reverting, the diff recalculates and reapplies, so hunk navigation stays accurate.

---

## 5. Reviewing Commit History

The sidebar supports a **commit history mode** that lets you browse past commits where Claude edited files. This is useful for reviewing what Claude did across multiple interactions.

### Entering history mode

With the sidebar open, press `H` to toggle commit history mode. The sidebar switches from showing current uncommitted changes to showing files from a past commit where Claude made edits.

### Navigating commits

| Key | Action |
|-----|--------|
| `H` | Toggle history mode on/off |
| `]g` | Jump to the next (newer) commit |
| `[g` | Jump to the previous (older) commit |

The sidebar header updates to show which commit you are viewing.

### Viewing commit diffs

When you open a file in history mode (press `<CR>` or `d` on a file), a new tab opens showing a diff of **commit~1 vs commit** -- that is, what changed in that specific commit. This lets you review exactly what Claude did in each past commit without affecting your working tree.

### Example workflow

1. Open the sidebar: `<leader>cs`
2. Press `H` to enter history mode
3. Use `[g` to go back to an earlier commit
4. Press `<CR>` on a file to see what Claude changed in that commit
5. Use `]g`/`[g` to browse through other commits
6. Press `H` again to return to the live view of uncommitted changes

---

## 6. Git HEAD Diffing (Key Concept)

This is the single most important concept in cc-watcher. Understanding it prevents confusion.

### How diffs work

cc-watcher compares files against `git show HEAD:<file>` to produce diffs. This is the primary comparison method because snapshots taken on `BufReadPost` would capture post-edit content (after Claude has already written the file), making them unreliable as a baseline.

Git HEAD provides a stable, known-good reference point that represents the last committed state of each file.

### How it works

1. Claude edits `src/handlers.rs`. The file watcher fires, the buffer reloads.
2. You activate the diff. It compares the current file against `git show HEAD:src/handlers.rs`.
3. The diff shows everything that changed since the last commit, which in the typical workflow is exactly what Claude did.

### Snapshot fallback for untracked files

For files that are **not tracked by git** (new files Claude created, files outside a git repository), cc-watcher falls back to in-memory snapshots. When you open a file, its content is stored as a snapshot. If git HEAD is not available, the diff compares against this snapshot instead.

### LRU cache

Snapshots (used as fallback) are stored in memory with an LRU eviction policy:

- **Max 100 files** -- when the 101st file is opened, the least-recently-accessed snapshot is evicted.
- **Max 10MB per file** -- files larger than 10MB are not snapshotted.

For most projects, 100 files is far more than you will have open in a session.

### Clearing snapshots

If you want to reset the snapshot baseline (relevant for untracked files):

```lua
require("cc-watcher.snapshots").clear()
```

This removes all snapshots. The next time you open a file, a new snapshot is taken. You can also remove a single file's snapshot:

```lua
require("cc-watcher.snapshots").remove(vim.fn.expand("%:p"))
```

---

## 7. Snacks Picker Integration

### Enable

```lua
require("cc-watcher").setup({
    integrations = {
        snacks = true,
    },
})
```

Requires [snacks.nvim](https://github.com/folke/snacks.nvim), which is included by default in LazyVim.

### Changed files picker

```vim
:ClaudeSnacks
```

Opens a snacks picker listing all files Claude has changed. Each entry shows:

- `●` / `○` indicator (live vs session)
- Relative file path (from git root)
- `+N/-M` stats

The **preview pane** shows a colored unified diff with green highlights for additions and red for deletions.

**Actions:**

| Key | Action |
|-----|--------|
| `<CR>` | Open file and activate inline diff (jumps to first change) |

### Hunks picker

```vim
:ClaudeSnacks hunks
```

Lists **every individual hunk** across all changed files. The preview shows the file content at the hunk location.

Pressing `<CR>` opens the file, jumps to the hunk line, centers the view, and activates the inline diff.

### Recommended keymaps

```lua
keys = {
    { "<leader>ct", "<cmd>ClaudeSnacks<cr>", desc = "Claude - changed files" },
    { "<leader>ch", "<cmd>ClaudeSnacks hunks<cr>", desc = "Claude - hunks" },
},
```

---

## 8. LazyVim Dashboard Setup

If you use LazyVim with the snacks.nvim dashboard (the default startup page), you can add cc-watcher entries so you can jump straight into reviewing Claude's changes when you open Neovim.

### Add to the startup page

Create `~/.config/nvim/lua/plugins/dashboard.lua`:

```lua
return {
  "snacks.nvim",
  opts = {
    dashboard = {
      preset = {
        -- stylua: ignore
        ---@type snacks.dashboard.Item[]
        keys = {
          { icon = " ", key = "f", desc = "Find File", action = ":lua Snacks.dashboard.pick('files')" },
          { icon = " ", key = "n", desc = "New File", action = ":ene | startinsert" },
          { icon = " ", key = "g", desc = "Find Text", action = ":lua Snacks.dashboard.pick('live_grep')" },
          { icon = " ", key = "r", desc = "Recent Files", action = ":lua Snacks.dashboard.pick('oldfiles')" },
          { icon = " ", key = "c", desc = "Config", action = ":lua Snacks.dashboard.pick('files', {cwd = vim.fn.stdpath('config')})" },
          { icon = " ", key = "s", desc = "Restore Session", section = "session" },
          { icon = " ", key = "w", desc = "Claude Changes", action = ":lua vim.cmd('enew'); vim.bo.bufhidden = 'wipe'; vim.cmd('ClaudeSnacks')" },
          { icon = " ", key = "x", desc = "Lazy Extras", action = ":LazyExtras" },
          { icon = "󰒲 ", key = "l", desc = "Lazy", action = ":Lazy" },
          { icon = " ", key = "q", desc = "Quit", action = ":qa" },
        },
      },
    },
  },
}
```

Pressing `w` on the startup page opens the snacks changed files picker with a colored diff preview. The `:enew` first dismisses the dashboard so you are left with a clean editor.

### Add statusline indicator

Create `~/.config/nvim/lua/plugins/lualine.lua`:

```lua
return {
  "nvim-lualine/lualine.nvim",
  opts = function(_, opts)
    table.insert(opts.sections.lualine_x, 1, {
      function()
        return require("cc-watcher").statusline()
      end,
      cond = function()
        local ok, watcher = pcall(require, "cc-watcher.watcher")
        return ok and vim.tbl_count(watcher.get_changed_files()) > 0
      end,
    })
  end,
}
```

This shows `󰚩 N` in the statusline when Claude has changed N files, and hides when there are no changes.

### Example workflow with dashboard

1. Claude Code is running in a tmux pane, editing files
2. Open Neovim — the dashboard appears
3. Press `w` — the snacks picker opens showing all files Claude changed with diffs
4. Browse the list, select a file — it opens with inline diff, cursor at the first change
5. Use `]c`/`[c` to navigate hunks, `cr` to revert any you do not want

---

## 9. trouble.nvim Integration

### Enable

```lua
require("cc-watcher").setup({
    integrations = {
        trouble = true,
    },
})
```

Requires [trouble.nvim](https://github.com/folke/trouble.nvim) **v3** to be installed.

### Usage

```vim
:ClaudeTrouble
```

Opens a trouble.nvim window with a diagnostic-style list of all changes Claude made. Each hunk appears as an item:

| Type | Meaning |
|------|---------|
| **Info** (blue) | Lines added |
| **Warning** (yellow) | Lines changed |
| **Hint** (green) | Lines deleted |

Each item shows the file path, line number, and a description like `+5/-2 lines (changed)`.

### Why use it

trouble.nvim gives you a bird's-eye view of every change in a single scrollable list, sorted by file and line number. It is particularly useful when Claude made many small changes across many files and you want to quickly scan all of them.

Click any item to jump directly to that hunk in the file.

---

## 10. Diffview Integration

### Enable

```lua
require("cc-watcher").setup({
    integrations = {
        diffview = true,
    },
})
```

This integration uses Neovim's built-in `diffthis` command for side-by-side diffs. It does **not** require the diffview.nvim plugin (despite the name).

### All changed files

```vim
:ClaudeDiffview
```

Opens a new tab with a side-by-side diff:

- **Left pane:** Snapshot content (pre-Claude state, readonly)
- **Right pane:** Current file content (editable)

Both panes are in vim diff mode, so you get the familiar `]c`/`[c` navigation and full syntax highlighting.

If Claude changed multiple files, navigate between them:

| Key | Action |
|-----|--------|
| `]f` | Next file |
| `[f` | Previous file |
| `q` | Close the diff tab |

A notification like `[1/3] src/handlers.rs` shows your position in the file list.

### Single file

```vim
:ClaudeDiffview src/handlers.rs
```

Opens a side-by-side diff for just that one file. Useful when you already know which file you want to review in detail.

### When to use it

Use `:ClaudeDiffview` when you want the most comprehensive, familiar diff experience. It works exactly like `vimdiff` -- you get full color-coded side-by-side comparison with scroll binding, fold management, and all the standard diff mode features.

For large changes spanning many files, this is the best way to do a thorough review.

---

## 11. Statusline Integration

cc-watcher provides a statusline component that shows the count of changed files.

### lualine.nvim

```lua
require("lualine").setup({
    sections = {
        lualine_x = {
            {
                require("cc-watcher").statusline,
                cond = function()
                    return require("cc-watcher").statusline() ~= ""
                end,
            },
        },
    },
})
```

This displays `"N"` (with the Claude icon) when there are changed files, and hides completely when there are none.

### heirline.nvim

```lua
local ClaudeStatus = {
    provider = function()
        local ok, ccw = pcall(require, "cc-watcher")
        if not ok then return "" end
        return ccw.statusline()
    end,
    condition = function()
        local ok, ccw = pcall(require, "cc-watcher")
        if not ok then return false end
        return ccw.statusline() ~= ""
    end,
    hl = { fg = "#cba6f7" },
}
```

Add `ClaudeStatus` to your statusline component tree wherever you want it to appear.

### What it shows

The statusline component returns an empty string when there are no changes (so it disappears), or a string like:

```
 3
```

where `3` is the number of files Claude has changed. The icon is U+F0EA9 (nerd font: Claude/bot icon).

---

## 12. Customization

### Overriding highlight groups

All highlight groups are defined with `default = true`, so your colorscheme or manual overrides take priority. Set them **after** loading your colorscheme:

```lua
-- Example: adapt for a light colorscheme
vim.api.nvim_set_hl(0, "ClaudeDiffAdd", { bg = "#d4edda", fg = "#155724" })
vim.api.nvim_set_hl(0, "ClaudeDiffChange", { bg = "#fff3cd", fg = "#856404" })
vim.api.nvim_set_hl(0, "ClaudeDiffDelete", { bg = "#f8d7da", fg = "#721c24" })
vim.api.nvim_set_hl(0, "ClaudeDiffAddSign", { fg = "#28a745" })
vim.api.nvim_set_hl(0, "ClaudeDiffChangeSign", { fg = "#ffc107" })
vim.api.nvim_set_hl(0, "ClaudeDiffDeleteSign", { fg = "#dc3545" })

-- Sidebar customization
vim.api.nvim_set_hl(0, "ClaudeHeader", { fg = "#6f42c1", bold = true })
vim.api.nvim_set_hl(0, "ClaudeActive", { fg = "#28a745" })
vim.api.nvim_set_hl(0, "ClaudeInactive", { fg = "#999999", italic = true })
```

Full list of highlight groups:

| Group | Default | Purpose |
|-------|---------|---------|
| `ClaudeDiffAdd` | green bg | Added lines in inline diff |
| `ClaudeDiffChange` | yellow bg | Changed lines in inline diff |
| `ClaudeDiffDelete` | red bg, dimmed | Deleted lines (virtual text) |
| `ClaudeDiffDeleteNr` | red fg | The `- ` / `~ ` prefix on virtual text lines |
| `ClaudeDiffAddSign` | green fg | Sign column: added |
| `ClaudeDiffChangeSign` | yellow fg | Sign column: changed |
| `ClaudeDiffDeleteSign` | red fg | Sign column: deleted |
| `ClaudeHeader` | mauve, bold | Sidebar title |
| `ClaudeActive` | green | "session active" text |
| `ClaudeInactive` | grey, italic | "no session" text |
| `ClaudeLive` | yellow | `●` indicator |
| `ClaudeSession` | blue | `○` indicator |
| `ClaudeFile` | light text | File names in sidebar |
| `ClaudeFileCurrent` | white, bold, bg | Currently open file in sidebar |
| `ClaudeFileLatest` | italic, DiagnosticWarn fg | Most recently edited file |
| `ClaudeStats` | dim | `+N -M` stats text |
| `ClaudeCount` | blue | Footer summary line |
| `ClaudeSep` | dark | Separator lines |
| `ClaudeHelp` | dim | "g? help" text |

### Changing keybindings

Disable the default global keymaps and set your own:

```lua
require("cc-watcher").setup({
    keys = {
        toggle_sidebar = false,  -- disable default <leader>cs
        toggle_diff = false,     -- disable default <leader>cd
    },
})

-- Set your own keybindings
vim.keymap.set("n", "<leader>cc", function()
    require("cc-watcher.sidebar").toggle()
end, { desc = "Claude sidebar" })

vim.keymap.set("n", "<leader>ci", function()
    require("cc-watcher.diff").show()
end, { desc = "Claude inline diff" })
```

The sidebar-local keybindings (`<CR>`, `d`, `o`, `H`, `]g`, `[g`, `r`, `q`, `g?`) and diff-local keybindings (`]c`, `[c`, `cr`) are always set on their respective buffers and cannot be changed through config. To override them, set up autocommands on the `claude-sidebar` filetype.

### Sidebar width

```lua
require("cc-watcher").setup({
    sidebar_width = 100,  -- default is 100 (use a fraction like 0.6 for 60%)
})
```

### Full customized example

```lua
{
    "elmomk/cc-watcher.nvim",
    event = { "BufReadPost", "BufNewFile" },
    cmd = {
        "ClaudeSidebar", "ClaudeDiff",
        "ClaudeSnacks", "ClaudeDiffview",
    },
    keys = {
        { "<leader>cc", function() require("cc-watcher.sidebar").toggle() end, desc = "Claude sidebar" },
        { "<leader>ci", function() require("cc-watcher.diff").show() end, desc = "Claude inline diff" },
        { "<leader>ct", "<cmd>ClaudeSnacks hunks<cr>", desc = "Claude hunks (snacks)" },
        { "<leader>cv", "<cmd>ClaudeDiffview<cr>", desc = "Claude diffview" },
    },
    config = function()
        require("cc-watcher").setup({
            sidebar_width = 100,
            keys = {
                toggle_sidebar = false,
                toggle_diff = false,
            },
            integrations = {
                snacks = true,
                diffview = true,
            },
        })

        -- Custom highlights for tokyonight
        vim.api.nvim_set_hl(0, "ClaudeHeader", { fg = "#bb9af7", bold = true })
        vim.api.nvim_set_hl(0, "ClaudeDiffAdd", { bg = "#1a2b1a" })
        vim.api.nvim_set_hl(0, "ClaudeDiffChange", { bg = "#2b2a1a" })
        vim.api.nvim_set_hl(0, "ClaudeDiffDelete", { bg = "#2b1a1a", fg = "#6a4050" })
    end,
}
```

---

## 13. Advanced Usage

### Lua API

cc-watcher exposes a programmatic API that you can use in your own scripts and configurations.

**Hook into change events:**

```lua
-- Called every time a file change is detected
require("cc-watcher.watcher").on_change(function(filepath, relpath)
    -- filepath: absolute path (e.g., "/home/user/project/src/main.rs")
    -- relpath: relative to cwd (e.g., "src/main.rs")
    print("Changed: " .. relpath)
end)
```

**Get all changed files:**

```lua
-- Returns a table: { [filepath] = true, ... }
local changed = require("cc-watcher.watcher").get_changed_files()
for filepath, _ in pairs(changed) do
    print(filepath)
end
```

**Check Claude session status:**

```lua
-- Returns session info table or nil
local session = require("cc-watcher.session").find_active_session(vim.fn.getcwd())
if session then
    print("Claude is running, PID: " .. session.pid)
end
```

**Get all files Claude edited in this session:**

```lua
require("cc-watcher.session").get_claude_edited_files_async(function(files)
    -- files: list of absolute file paths
    for _, filepath in ipairs(files) do
        print(filepath)
    end
end)
```

**Snapshot operations:**

```lua
local snap = require("cc-watcher.snapshots")

-- Check if a snapshot exists
snap.has("/path/to/file.rs")  -- true/false

-- Get snapshot data
local data = snap.get("/path/to/file.rs")
-- data.lines: table of lines
-- data.raw: raw string content
-- data.mtime: modification time when snapshot was taken

-- Take a new snapshot (overwrites existing)
snap.take("/path/to/file.rs")

-- Remove one snapshot
snap.remove("/path/to/file.rs")

-- Clear all snapshots
snap.clear()

-- Count stored snapshots
snap.count()  -- number
```

**Diff operations:**

```lua
local diff = require("cc-watcher.diff")

-- Get add/delete stats for a file
local add, del = diff.file_stats("/path/to/file.rs", bufnr)

-- Count hunks
local count = diff.hunk_count("/path/to/file.rs", bufnr)

-- Programmatically show/toggle diff
diff.show("/path/to/file.rs")

-- Clear diff from a buffer
diff.clear(bufnr)

-- Apply sign-column-only indicators
diff.apply_signs(bufnr, "/path/to/file.rs")
```

### Example: auto-run tests when Claude edits test files

```lua
require("cc-watcher.watcher").on_change(function(filepath, relpath)
    if relpath:match("_test%.") or relpath:match("_spec%.") then
        vim.notify("Test file changed, running tests...", vim.log.levels.INFO)
        vim.cmd("!cargo test 2>&1 | tail -5")
    end
end)
```

### Example: auto-show diff when Claude changes the current buffer

```lua
require("cc-watcher.watcher").on_change(function(filepath, relpath)
    local current = vim.api.nvim_buf_get_name(0)
    if filepath == current then
        vim.defer_fn(function()
            require("cc-watcher.diff").show(filepath)
        end, 200)  -- small delay to let the buffer reload
    end
end)
```

### Example: notification with file details

```lua
require("cc-watcher.watcher").on_change(function(filepath, relpath)
    local bufnr = vim.fn.bufnr(filepath)
    if bufnr ~= -1 then
        local add, del = require("cc-watcher.diff").file_stats(filepath, bufnr)
        vim.notify(
            string.format("Claude edited %s (+%d/-%d)", relpath, add, del),
            vim.log.levels.INFO
        )
    end
end)
```

### Listening for JSONL updates

```lua
-- Called when Claude's session log file changes on disk
require("cc-watcher.session").on_jsonl_change(function(jsonl_path)
    -- Useful for triggering custom refresh logic
    print("Session log updated: " .. jsonl_path)
end)
```

---

## 14. Lazy Loading

cc-watcher supports three lazy loading strategies. Each has trade-offs.

### Event-based (recommended)

```lua
event = { "BufReadPost", "BufNewFile" }
```

The plugin loads when you open any file. This is the recommended approach because it ensures **snapshots are captured early** for untracked files and file watchers are registered promptly. Diffs for git-tracked files always use git HEAD regardless of when the plugin loads.

### Command-based

```lua
cmd = { "ClaudeSidebar", "ClaudeDiff", "ClaudeSnacks", "ClaudeTrouble", "ClaudeDiffview" }
```

The plugin loads on first command invocation. Startup is faster, but **untracked files opened before your first command will not have snapshots**. Git-tracked files are unaffected. Acceptable if you always open the sidebar early in your session.

### Key-based

```lua
keys = {
    { "<leader>cs", function() require("cc-watcher")._ensure_setup(); require("cc-watcher.sidebar").toggle() end, desc = "Claude sidebar" },
    { "<leader>cd", function() require("cc-watcher")._ensure_setup(); require("cc-watcher.diff").show() end, desc = "Claude inline diff" },
}
```

Same trade-off as command-based: loads on first keypress.

### Recommended: combine all three

For the best experience, combine event + cmd + keys. The event trigger ensures snapshots are captured, while cmd and keys provide nice lazy.nvim completions and which-key hints:

```lua
{
    "elmomk/cc-watcher.nvim",
    event = { "BufReadPost", "BufNewFile" },
    cmd = {
        "ClaudeSidebar", "ClaudeDiff",
        "ClaudeSnacks", "ClaudeTrouble", "ClaudeDiffview",
    },
    keys = {
        { "<leader>cs", desc = "Claude - toggle sidebar" },
        { "<leader>cd", desc = "Claude - toggle inline diff" },
    },
    config = function()
        require("cc-watcher").setup()
    end,
}
```

The plugin also exports a helper for lazy.nvim specs:

```lua
local ccw_lazy = require("cc-watcher").lazy
-- ccw_lazy.cmd, ccw_lazy.keys, ccw_lazy.event are pre-defined
```

### Caveat

If you use pure `cmd` or `keys` loading (without `event`), any file you open **before** the first trigger will not have a snapshot. For git-tracked files this is not an issue since git HEAD is always used as the baseline. For untracked files, no snapshot means no diff baseline will be available until the plugin loads.

---

## 15. Tips and Tricks

**Quick file-by-file review:**
Open the sidebar (`:ClaudeSidebar`), navigate to each file with `j`/`k`, press `d` to open with diff. Review the hunks with `]c`/`[c`. Revert anything bad with `cr`. Move to the next file.

**Rapid overview of all changes:**
`:ClaudeSnacks hunks` gives you a flat list of every hunk across the entire project. Scan the list, jump to anything interesting.

**Selective revert:**
Use `cr` (revert hunk) to cherry-pick which changes to keep. If Claude added 5 hunks and one of them is wrong, navigate to it and `cr`. The other 4 hunks stay intact.

**Leave the sidebar open:**
The sidebar auto-updates as Claude works. Leave it open and glance at it periodically. The `+N/-M` stats tell you how substantial the changes are without having to open each file.

**Statusline for passive awareness:**
Add the statusline component so you always know how many files Claude has changed, even without the sidebar open.

**Review past Claude commits:**
Press `H` in the sidebar to enter history mode and browse through past commits where Claude made changes. Use `]g`/`[g` to step through commits. Opening a file shows the commit diff in a new tab.

**Diffview for large reviews:**
When Claude makes extensive changes across many files, `:ClaudeDiffview` gives you a traditional side-by-side view that is easiest to read for big diffs. Use `]f`/`[f` to flip between files.

**Reset after accepting changes:**
After you have reviewed and are happy with Claude's work, clear snapshots to reset the baseline:

```lua
require("cc-watcher.snapshots").clear()
```

Now any future changes will be diffed against the current state.

**Combine with which-key:**
cc-watcher automatically registers the `<leader>c` group with which-key if it is installed, showing it as "Claude Code". Your custom keymaps under `<leader>c` will appear in the which-key popup.

---

## 16. Troubleshooting

### "no session" in the sidebar

The plugin cannot find an active Claude Code process for the current working directory.

**Check:**
- Is Claude Code running? Check with `ps aux | grep claude`.
- Is it running in the **same directory**? The plugin matches sessions by `cwd`. If Claude was started in `/home/user/project` but Neovim is in `/home/user/project/src`, they will not match.
- Session files are at `~/.claude/sessions/`. List them to see what is there:

```bash
ls -la ~/.claude/sessions/
cat ~/.claude/sessions/*.json  # inspect the cwd and pid fields
```

### "No snapshot" warnings

A file was changed that is not tracked by git, and no snapshot exists. Diffs normally compare against git HEAD. For untracked files, the plugin falls back to in-memory snapshots.

**Solutions:**
- For git-tracked files, no action needed -- git HEAD is always used as the baseline.
- For untracked files, open them in Neovim before Claude edits them so a snapshot is captured.
- Use the `event = { "BufReadPost", "BufNewFile" }` lazy loading strategy so snapshots are captured automatically when files are opened.

### Changes not appearing in the sidebar

**Check JSONL files:**

```bash
ls -la ~/.claude/projects/
# Find the directory for your project, then:
ls -la ~/.claude/projects/*/
```

The plugin needs a `.jsonl` file whose first line contains your project's absolute path. If no matching JSONL exists, session detection will not work.

**Try refreshing:** Press `r` in the sidebar, or close and reopen it.

**Check live detection:** If you have the file open in a buffer, edits should be detected via `fs_event`. If they are not, check that `vim.o.autoread` is `true` (cc-watcher sets this automatically).

### Integration commands not working

```
cc-watcher: snacks integration is disabled. Enable it with integrations.snacks = true
```

This means the integration is not enabled in your config. Add it:

```lua
require("cc-watcher").setup({
    integrations = {
        snacks = true,
    },
})
```

If you see a "not found" error, the dependency itself is not installed. Install it separately.

### Statusline showing nothing

The statusline component returns an empty string when there are no changed files. It also returns empty if the plugin has not been initialized yet. Make sure:

1. The plugin is loaded (check with `:lua print(vim.g.loaded_cc_watcher)`).
2. Claude has actually made changes.
3. Test it directly: `:lua print(require("cc-watcher").statusline())`.

### Diff shows unexpected changes

If the diff includes changes you made (not Claude), this is expected -- diffs compare against git HEAD, so any uncommitted changes (yours and Claude's) will appear. To see only Claude's changes, commit your own work before asking Claude to edit files.

For untracked files that use snapshot fallback, you can reset the baseline:

```lua
require("cc-watcher.snapshots").clear()
```

Then reopen the file. A new snapshot is taken. Future diffs will be relative to this point.

### Performance concerns

cc-watcher uses `libuv fs_event` watchers that consume zero CPU when idle. JSONL parsing is incremental (only new bytes are read). Sidebar refreshes are debounced (300ms for JSONL changes, 500ms for batched notifications). Snapshots use an LRU cache capped at 100 entries.

If you notice lag, check how many files are in the sidebar. A very large number of changed files (hundreds) could slow down sidebar rendering since stats are computed for each loaded buffer.
