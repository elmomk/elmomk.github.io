# Building Neovim Plugins in Lua: A Practical Guide

*Using cc-watcher.nvim as a real-world reference*

This guide walks you through every aspect of building a Neovim plugin in Lua.
Rather than inventing toy examples, it draws from the cc-watcher.nvim codebase
-- a plugin that watches for file changes, renders a sidebar, computes inline
diffs, and integrates with half a dozen other plugins. By the time you finish
this guide, you will understand how all of those pieces fit together and be able
to build your own plugin from scratch.

**Prerequisites:** You should be comfortable reading Lua (tables, functions,
metatables are not mysteries to you) and have a working Neovim 0.10+ setup.
You do not need prior experience writing Neovim plugins.

---

## Table of Contents

1. [Plugin Structure](#1-plugin-structure)
2. [Entry Points](#2-entry-points)
3. [Module System](#3-module-system)
4. [Configuration](#4-configuration)
5. [Autocommands](#5-autocommands)
6. [Keymaps](#6-keymaps)
7. [Working with Buffers](#7-working-with-buffers)
8. [Extmarks and Highlights](#8-extmarks-and-highlights)
9. [Windows and Splits](#9-windows-and-splits)
10. [File I/O with libuv](#10-file-io-with-libuv)
11. [Shell Commands](#11-shell-commands)
12. [Integrating with Other Plugins](#12-integrating-with-other-plugins)
13. [Debugging](#13-debugging)
14. [Testing](#14-testing)
15. [Publishing](#15-publishing)

---

## 1. Plugin Structure

Every Neovim Lua plugin follows a convention-over-configuration directory
layout. Neovim looks for specific directories in your plugin's root, and
each has a well-defined purpose.

### The Standard Layout

```
my-plugin.nvim/
  lua/
    my-plugin/
      init.lua        -- Main module: require("my-plugin") loads this
      watcher.lua     -- Sub-module: require("my-plugin.watcher")
      sidebar.lua     -- Sub-module: require("my-plugin.sidebar")
      diff.lua        -- Sub-module: require("my-plugin.diff")
      highlights.lua  -- Sub-module: require("my-plugin.highlights")
      util.lua        -- Shared helpers
  plugin/
    my-plugin.lua     -- Auto-loaded entry point (commands, guard)
  doc/
    my-plugin.txt     -- Vimdoc help file (:help my-plugin)
  tests/
    minimal_init.lua  -- Minimal config for headless test runner
    test_init.lua     -- Tests for init module
    test_watcher.lua  -- Tests for watcher module
  Makefile            -- Test runner
  README.md
```

Here is the actual cc-watcher.nvim layout:

```
cc-watcher.nvim/
  lua/
    cc-watcher/
      init.lua          -- Setup, config, statusline component
      watcher.lua       -- Per-file fs_event watchers
      session.lua       -- Read Claude Code session JSONL data
      sidebar.lua       -- Sidebar UI (buffer, window, rendering)
      diff.lua          -- Inline diff highlighting, hunk navigation
      highlights.lua    -- Highlight group definitions
      snapshots.lua     -- LRU file content cache
      snacks.lua        -- snacks.nvim picker integration
      trouble.lua       -- trouble.nvim source
      util.lua          -- Shared helpers (relpath, file I/O, diff)
    trouble/
      sources/
        claude.lua      -- Proxy: trouble auto-discovers this path
  plugin/
    cc-watcher.lua      -- Guard, user command registration
  tests/
    minimal_init.lua
    test_init.lua
    test_session.lua
    test_snapshots.lua
    test_trouble.lua
    test_watcher.lua
  doc/
    cc-watcher.txt
  Makefile
```

### What Goes Where

**`lua/`** -- All your Lua source code. Neovim adds this to the Lua
`package.path` so that `require("cc-watcher.watcher")` finds
`lua/cc-watcher/watcher.lua`. This is the heart of your plugin.

**`plugin/`** -- Scripts that run automatically when Neovim starts (or when a
plugin manager loads the plugin). Keep this minimal: a load guard, user
commands, and nothing else. Heavy initialization goes in `setup()`.

**`doc/`** -- Vimdoc help files (`.txt`) and optionally a tags file. Run
`:helptags doc/` to generate tags. Markdown tutorials can also live here.

**`tests/`** -- Not a Neovim convention per se, but the standard place for test
files. The test runner is typically invoked via a Makefile.

### Naming Conventions

Plugin names conventionally use hyphens (`cc-watcher.nvim`) while Lua module
names map hyphens to the filesystem path (`lua/cc-watcher/init.lua`). Lua
`require()` uses dots as separators: `require("cc-watcher.sidebar")`.

If your plugin name contains hyphens, the directory under `lua/` must match
exactly. Lua has no trouble with hyphenated directory names.

---

## 2. Entry Points

### The plugin/ File and the Guard Pattern

The file in `plugin/` runs every time Neovim starts. Its job is to register
user commands and set a global flag so it never runs twice. Here is the actual
cc-watcher.nvim entry point from `plugin/cc-watcher.lua`:

```lua
-- cc-watcher.nvim -- auto-loaded entry point
-- Registers lightweight command stubs so lazy.nvim can load on cmd = { ... }

if vim.g.loaded_cc_watcher then return end
vim.g.loaded_cc_watcher = true
```

The guard pattern (`if vim.g.loaded_X then return end`) prevents double
execution. This matters because:

- A user might source their config twice.
- Plugin managers may load the plugin file at different stages.
- Without the guard, you would register duplicate commands and autocommands.

Always use a unique global variable. The convention is
`vim.g.loaded_<plugin_name>` with underscores replacing hyphens.

### User Commands via `nvim_create_user_command`

After the guard, register your commands. Each command should be a thin wrapper
that calls into your Lua modules:

```lua
local function ensure()
    require("cc-watcher")._ensure_setup()
end

vim.api.nvim_create_user_command("ClaudeSidebar", function()
    ensure()
    require("cc-watcher.sidebar").toggle()
end, {
    desc = "Toggle Claude Code changed files sidebar",
})

vim.api.nvim_create_user_command("ClaudeDiff", function()
    ensure()
    require("cc-watcher.diff").show()
end, {
    desc = "Toggle inline diff for current file",
})
```

Key points:

- **`ensure()`** calls `_ensure_setup()`, which runs `setup()` with defaults
  if the user never called it. This makes commands work even without explicit
  configuration.
- **`require()` inside the callback** means the module is loaded lazily -- only
  when the user actually runs the command.
- **`desc`** is important: it shows up in `:command`, which-key, and
  telescope/fzf command pickers.

### Commands with Arguments and Completion

cc-watcher registers several commands that accept optional arguments:

```lua
vim.api.nvim_create_user_command("ClaudeSnacks", function(args)
    ensure()
    local cfg = require("cc-watcher").config
    if not cfg.integrations.snacks then
        vim.notify(
            "cc-watcher: snacks integration is disabled. "
            .. "Enable it with integrations.snacks = true",
            vim.log.levels.WARN
        )
        return
    end
    local ok, snacks_mod = pcall(require, "cc-watcher.snacks")
    if not ok then
        vim.notify("cc-watcher: snacks.nvim not found", vim.log.levels.ERROR)
        return
    end
    local sub = args.fargs[1]
    if sub == "hunks" then snacks_mod.hunks()
    else snacks_mod.changed_files() end
end, {
    nargs = "?",
    complete = function() return { "changed_files", "hunks" } end,
    desc = "Snacks: Claude Code changes",
})
```

The key options:

- **`nargs = "?"`** -- zero or one argument. Use `"*"` for any number, `"+"` for one or more, or a number for an exact count.
- **`complete`** -- a function (or string like `"file"`, `"buffer"`) for tab completion. Return a list of strings.
- **`args.fargs`** -- the parsed argument list (split on whitespace). `args.args` gives the raw string.

For file path completion, you can use the built-in `"file"` completer:

```lua
vim.api.nvim_create_user_command("ClaudeDiffview", function(args)
    ensure()
    -- ... validation ...
    local filepath = args.fargs[1]
    if filepath then
        dv.open_file(vim.fn.fnamemodify(filepath, ":p"))
    else
        dv.open()
    end
end, {
    nargs = "?",
    complete = "file",
    desc = "Diffview: Claude Code changes",
})
```

---

## 3. Module System

### The M Pattern

Every cc-watcher module follows the same structure:

```lua
local M = {}

-- Private state (module-local, not exposed)
local some_internal_state = {}

-- Public function
function M.do_something()
    -- ...
end

-- Private function (local, not on M)
local function helper()
    -- ...
end

return M
```

The table `M` is your module's public API. Anything attached to `M` is
accessible to callers via `require()`. Local variables and functions are private.

This pattern is idiomatic in the Neovim ecosystem. Here is how `watcher.lua`
uses it:

```lua
local M = {}

local file_watchers = {}       -- private: per-file libuv handles
local changed_files = {}       -- private: set of changed file paths
local on_change_callbacks = {} -- private: registered callback list

function M.on_change(cb)          -- public: register a callback
    on_change_callbacks[#on_change_callbacks + 1] = cb
end

function M.get_changed_files()    -- public: read-only accessor
    return changed_files
end

function M.mark_changed(filepath) -- public: mark a file as changed
    -- ...fires callbacks...
end

return M
```

### require() and Caching

Lua's `require()` caches modules in `package.loaded`. The first call to
`require("cc-watcher.sidebar")` executes the file and stores the result.
Subsequent calls return the cached table instantly.

This has an important implication: **module-level code runs exactly once.** In
`sidebar.lua`, this line runs once when the module is first required:

```lua
local watcher = require("cc-watcher.watcher")
```

After that, `watcher` is a reference to the watcher module's `M` table and
stays valid for the lifetime of the Neovim session.

### Lazy Loading Modules

You can defer `require()` calls until they are needed. cc-watcher's
`init.lua` does this inside `setup()`:

```lua
function M.setup(opts)
    M.config = vim.tbl_deep_extend("force", defaults, opts or {})

    if _setup_done then
        return
    end
    _setup_done = true

    local watcher = require("cc-watcher.watcher")
    local sidebar = require("cc-watcher.sidebar")
    local session = require("cc-watcher.session")
    local diff = require("cc-watcher.diff")

    watcher.setup()
    sidebar.setup()
    diff.setup()
    session.watch_jsonl()
    -- ...
end
```

The four `require()` calls happen inside `setup()`, not at the top of the file.
This means those modules are not loaded until the user (or plugin manager) calls
`setup()`. For plugins loaded via `lazy.nvim` with `event` or `cmd` triggers,
this keeps startup fast.

An even more aggressive form of lazy loading is to `require()` inside a
callback:

```lua
vim.keymap.set("n", keys.toggle_sidebar, function()
    require("cc-watcher.sidebar").toggle()
end, { silent = true, desc = "Claude - toggle sidebar" })
```

The sidebar module is not loaded until the user presses the key for the first
time.

### Cross-Module Dependencies

Modules can `require()` each other freely because of Lua's caching. However,
beware of **circular requires** -- if module A requires module B at the top
level and module B requires module A at the top level, one of them will get an
incomplete table.

cc-watcher avoids this by:

1. Having a clear dependency hierarchy (util and snapshots at the bottom,
   init at the top).
2. Using `require()` inside functions rather than at module scope when there
   is any risk of circularity. For example, `util.lua` requires `snapshots`
   inside `get_old_text()`, not at the top of the file:

```lua
function M.get_old_text(filepath, cwd, current_text)
    -- ...
    local snapshots = require("cc-watcher.snapshots")
    local snap = snapshots.get(filepath)
    -- ...
end
```

---

## 4. Configuration

### The Defaults Table and `vim.tbl_deep_extend`

Every good plugin provides sensible defaults and lets users override them.
Here is the pattern from cc-watcher's `init.lua`:

```lua
local defaults = {
    sidebar_width = 36,
    keys = {
        toggle_sidebar = "<leader>cs",
        toggle_diff = "<leader>cd",
    },
    integrations = {
        snacks = false,
        fzf_lua = false,
        trouble = false,
        diffview = false,
    },
}

M.config = vim.deepcopy(defaults)
```

`vim.deepcopy(defaults)` creates an independent copy so that `M.config` can be
mutated without affecting the `defaults` table. This is important because
`defaults` serves as the baseline for every `setup()` call.

### The setup() Function

```lua
function M.setup(opts)
    M.config = vim.tbl_deep_extend("force", defaults, opts or {})

    if _setup_done then
        return
    end
    _setup_done = true

    -- One-time initialization: register watchers, keymaps, etc.
end
```

`vim.tbl_deep_extend("force", defaults, opts or {})` does a recursive merge.
The user only needs to specify what they want to change:

```lua
-- User's config: only override what differs from defaults
require("cc-watcher").setup({
    sidebar_width = 42,
    integrations = { snacks = true },
})
-- Result: sidebar_width = 42, keys.toggle_sidebar = "<leader>cs" (preserved),
-- integrations.snacks = true, integrations.trouble = false (preserved)
```

The three merge strategies for `vim.tbl_deep_extend`:

- `"force"` -- right table wins on conflict (most common for setup)
- `"keep"` -- left table wins on conflict
- `"error"` -- throw on conflict

### Idempotent Setup

Notice the `_setup_done` guard. The first call to `setup()` does full
initialization (registering autocommands, keymaps, watchers). Subsequent
calls update the config but skip initialization. This prevents duplicate
autocommands and keymaps.

```lua
local _setup_done = false

function M.setup(opts)
    M.config = vim.tbl_deep_extend("force", defaults, opts or {})

    if _setup_done then
        return  -- Config updated, but no re-initialization
    end
    _setup_done = true

    -- Heavy one-time setup here
end
```

### Ensuring Setup Runs

What if the user never calls `setup()` but runs a command like
`:ClaudeSidebar`? cc-watcher handles this with `_ensure_setup()`:

```lua
function M._ensure_setup()
    if not _setup_done then
        M.setup()  -- Called with no args = all defaults
    end
end
```

The `plugin/cc-watcher.lua` commands call this before doing anything:

```lua
vim.api.nvim_create_user_command("ClaudeSidebar", function()
    require("cc-watcher")._ensure_setup()
    require("cc-watcher.sidebar").toggle()
end, { desc = "Toggle Claude Code changed files sidebar" })
```

### Integration with lazy.nvim `opts`

When using lazy.nvim, the `opts` table is passed directly to `setup()`:

```lua
-- In user's lazy.nvim config:
{
    "author/cc-watcher.nvim",
    opts = {
        sidebar_width = 42,
        integrations = { snacks = true },
    },
}
-- lazy.nvim will call: require("cc-watcher").setup(opts)
```

cc-watcher also exports a `lazy` table with trigger hints so that lazy.nvim
knows when to load the plugin:

```lua
M.lazy = {
    cmd = {
        "ClaudeSidebar", "ClaudeDiff",
        "ClaudeSnacks", "ClaudeFzf", "ClaudeTrouble", "ClaudeDiffview",
    },
    keys = {
        { "<leader>cs", function() ... end, desc = "Claude - toggle sidebar" },
        { "<leader>cd", function() ... end, desc = "Claude - toggle inline diff" },
    },
    event = { "BufReadPost", "BufNewFile" },
}
```

A user can spread this into their spec:

```lua
{ "author/cc-watcher.nvim", opts = { ... }, unpack(require("cc-watcher").lazy) }
```

---

## 5. Autocommands

Autocommands let your plugin react to events in the editor. They are
fundamental to plugins that need to track buffer state.

### Creating Autogroups

Always create an augroup for your autocommands. This lets you clear them when
the plugin reloads and prevents duplicates:

```lua
local augroup = vim.api.nvim_create_augroup("ClaudeCodeWatcher", { clear = true })
```

`{ clear = true }` removes any existing autocommands in this group before
creating new ones. If your plugin is re-sourced during development, this
prevents duplicate handlers.

### Registering Autocommands

Here is how `watcher.lua` sets up file watching:

```lua
vim.api.nvim_create_autocmd({ "BufReadPost", "BufNewFile" }, {
    group = augroup,
    callback = function(args)
        if vim.bo[args.buf].buftype ~= "" then return end
        local filepath = vim.api.nvim_buf_get_name(args.buf)
        if filepath == "" then return end
        if not snapshots.has(filepath) then
            snapshots.take(filepath)
        end
        watch_file(filepath)
    end,
})
```

The anatomy:

- **First argument** -- a string or table of event names. `{ "BufReadPost", "BufNewFile" }` fires for both events.
- **`group`** -- the augroup. Always set this.
- **`callback`** -- a Lua function that receives an `args` table.

The `args` table contains:

- `args.buf` -- the buffer number that triggered the event
- `args.event` -- which event fired (useful when listening to multiple)
- `args.match` -- the file pattern that matched
- `args.file` -- the filename (for file-related events)

### Common Events and Patterns

**Reacting to buffer changes from external tools:**

```lua
-- Detect when a file is modified outside Neovim
vim.api.nvim_create_autocmd("FileChangedShellPost", {
    group = augroup,
    callback = function(args)
        local filepath = vim.api.nvim_buf_get_name(args.buf)
        if filepath ~= "" and snapshots.has(filepath) then
            M.mark_changed(filepath)
        end
    end,
})
```

**Refreshing UI when the user switches buffers:**

```lua
-- Re-render sidebar when switching buffers to update current file highlight
vim.api.nvim_create_autocmd("BufEnter", {
    group = augroup,
    callback = function()
        if is_open() then
            vim.schedule(function() M.render() end)
        end
    end,
})
```

**Polling on focus gain (fallback for systems without fs_event):**

```lua
vim.api.nvim_create_autocmd({ "FocusGained", "BufEnter" }, {
    group = augroup,
    callback = function()
        if vim.fn.getcmdwintype() ~= "" then return end
        local filepath = vim.api.nvim_buf_get_name(0)
        if filepath ~= "" and not file_watchers[filepath] then
            vim.cmd("checktime")
        end
    end,
})
```

Note the guard `vim.fn.getcmdwintype() ~= ""` -- this prevents running
`checktime` while the command-line window is open, which would cause an error.

**Cleanup on buffer deletion:**

```lua
vim.api.nvim_create_autocmd({ "BufDelete", "BufUnload", "BufWipeout" }, {
    group = augroup,
    callback = function(args)
        local filepath = vim.api.nvim_buf_get_name(args.buf)
        if filepath ~= "" then unwatch_file(filepath) end
    end,
})
```

**Handling buffer rename (`:saveas`):**

```lua
vim.api.nvim_create_autocmd("BufFilePost", {
    group = augroup,
    callback = function(args)
        local new_path = vim.api.nvim_buf_get_name(args.buf)
        if new_path ~= "" then
            if not snapshots.has(new_path) then
                snapshots.take(new_path)
            end
            watch_file(new_path)
        end
    end,
})
```

### Filtering Autocommands

You can restrict autocommands to specific patterns or buffer types:

```lua
-- Only fire for Lua files
vim.api.nvim_create_autocmd("BufWritePost", {
    group = augroup,
    pattern = "*.lua",
    callback = function(args)
        -- Reload module
    end,
})

-- Only fire in specific buffers
vim.api.nvim_create_autocmd("BufWipeout", {
    group = augroup,
    callback = function(args)
        active_diffs[args.buf] = nil
    end,
})
```

### The `vim.schedule` and `vim.schedule_wrap` Patterns

Some Neovim APIs cannot be called from certain contexts (libuv callbacks,
fast events). Wrap them with `vim.schedule`:

```lua
-- Inside a libuv fs_event callback:
handle:start(filepath, {}, vim.schedule_wrap(function(err, _, events)
    if err then return end
    -- Safe to call Neovim APIs here because vim.schedule_wrap defers
    -- the execution to the main event loop
    M.mark_changed(filepath)
end))
```

`vim.schedule(fn)` queues a function for safe execution on the main loop.
`vim.schedule_wrap(fn)` returns a new function that, when called, schedules
`fn`. Use `schedule_wrap` for callbacks you pass to libuv or external code.

---

## 6. Keymaps

### Global Keymaps

Set global keymaps in your `setup()` function, guarded by config:

```lua
local keys = M.config.keys
if keys.toggle_sidebar then
    vim.keymap.set("n", keys.toggle_sidebar, function()
        require("cc-watcher.sidebar").toggle()
    end, {
        silent = true, desc = "Claude - toggle sidebar",
    })
end
if keys.toggle_diff then
    vim.keymap.set("n", keys.toggle_diff, function()
        require("cc-watcher.diff").show()
    end, {
        silent = true, desc = "Claude - toggle inline diff",
    })
end
```

Key points:

- **`vim.keymap.set(mode, lhs, rhs, opts)`** is the modern API. Avoid the
  older `vim.api.nvim_set_keymap`.
- **`silent = true`** suppresses command echo.
- **`desc`** is critical for discoverability (which-key, telescope keymaps).
- **Configurable LHS** -- let users set `keys.toggle_sidebar = false` to
  disable a keymap entirely.
- **Lazy require in the callback** -- the module is loaded on first keypress.

### Buffer-Local Keymaps

When you open a special buffer (like a sidebar or diff view), set keymaps
that only apply to that buffer:

```lua
-- From sidebar.lua:
local opts = { buffer = sidebar_buf, nowait = true, silent = true }

vim.keymap.set("n", "<CR>", open_with_diff, opts)
vim.keymap.set("n", "d",    open_with_diff, opts)
vim.keymap.set("n", "o",    open_with_diff, opts)
vim.keymap.set("n", "r",    function() M.render() end, opts)
vim.keymap.set("n", "q",    function() M.close() end, opts)
vim.keymap.set("n", "g?",   show_help, opts)
```

- **`buffer = sidebar_buf`** makes these keymaps local to the sidebar buffer.
  They vanish when the buffer is wiped.
- **`nowait = true`** means Neovim does not wait for additional keys. Without
  this, pressing `d` would wait to see if you type `dd`, `dw`, etc.

### Temporary Keymaps (Diff Navigation)

`diff.lua` sets up hunk navigation keymaps that only exist while a diff is
active, and removes them when the diff is cleared:

```lua
-- Setting keymaps when diff is shown:
vim.keymap.set("n", "]c", function()
    local row = vim.api.nvim_win_get_cursor(0)[1]
    for _, h in ipairs(hunk_lines) do
        if h > row then
            vim.api.nvim_win_set_cursor(0, { h, 0 })
            vim.cmd("normal! zz")
            flash_line(bufnr, h)
            return
        end
    end
    vim.notify("No next hunk", vim.log.levels.INFO)
end, { buffer = bufnr, silent = true, desc = "Next Claude hunk" })

-- Cleanup when diff is cleared:
local function clear(bufnr)
    vim.api.nvim_buf_clear_namespace(bufnr, ns, 0, -1)
    vim.api.nvim_buf_clear_namespace(bufnr, sign_ns, 0, -1)
    pcall(vim.keymap.del, "n", "]c", { buffer = bufnr })
    pcall(vim.keymap.del, "n", "[c", { buffer = bufnr })
    pcall(vim.keymap.del, "n", "cr", { buffer = bufnr })
    active_diffs[bufnr] = nil
end
```

Note the `pcall` around `vim.keymap.del` -- if the keymap was already removed
(e.g., buffer was wiped), this prevents an error.

### Integrating with which-key

If which-key.nvim is installed, register group labels so your keymaps appear
in a nice hierarchy:

```lua
local wk_ok, wk = pcall(require, "which-key")
if wk_ok and wk.add then
    pcall(wk.add, {
        { "<leader>c", group = "Claude Code" },
    })
end
```

Use `pcall` for the require so your plugin works without which-key. The
extra `pcall` around `wk.add` protects against API changes between
which-key versions.

---

## 7. Working with Buffers

### Creating Scratch Buffers

A "scratch" buffer is a non-file buffer used for UI elements like sidebars,
help popups, or preview panes. Here is how `sidebar.lua` creates one:

```lua
sidebar_buf = vim.api.nvim_create_buf(false, true)
-- Args: (listed, scratch)
-- listed = false: does not appear in :ls
-- scratch = true: sets buftype, noswapfile, etc.
```

After creating it, set buffer options:

```lua
vim.bo[sidebar_buf].buftype = "nofile"   -- Not associated with a file
vim.bo[sidebar_buf].bufhidden = "wipe"   -- Destroy when hidden
vim.bo[sidebar_buf].swapfile = false     -- No swap file
vim.bo[sidebar_buf].filetype = "claude-sidebar"  -- Custom filetype
```

Common `buftype` values:

- `"nofile"` -- buffer is not associated with a file
- `"nowrite"` -- buffer cannot be written
- `"prompt"` -- buffer for user input prompts
- `""` -- normal file buffer (default)

Common `bufhidden` values:

- `"wipe"` -- completely destroy buffer when it is no longer displayed
- `"hide"` -- hide but keep buffer data (default)
- `"delete"` -- delete buffer when hidden
- `"unload"` -- unload buffer contents when hidden

### Reading and Writing Buffer Lines

```lua
-- Read all lines from a buffer:
local lines = vim.api.nvim_buf_get_lines(bufnr, 0, -1, false)
-- Args: (buffer, start_line, end_line, strict_indexing)
-- 0 = first line, -1 = last line, false = don't error on out-of-range

-- Write lines to a buffer:
vim.bo[sidebar_buf].modifiable = true
vim.api.nvim_buf_set_lines(sidebar_buf, 0, -1, false, lines)
vim.bo[sidebar_buf].modifiable = false
```

The pattern of toggling `modifiable` is important for read-only buffers. You
lock the buffer after writing to prevent the user from editing the sidebar
content.

### Getting Buffer Metadata

```lua
-- Get the buffer's file path:
local filepath = vim.api.nvim_buf_get_name(bufnr)

-- Check if a buffer is loaded:
vim.api.nvim_buf_is_loaded(bufnr)

-- Check buffer type (skip special buffers):
if vim.bo[args.buf].buftype ~= "" then return end

-- List all loaded buffers:
for _, bufnr in ipairs(vim.api.nvim_list_bufs()) do
    if vim.api.nvim_buf_is_loaded(bufnr) then
        local name = vim.api.nvim_buf_get_name(bufnr)
        -- ...
    end
end
```

### Replacing Lines in a Range (Hunk Revert)

`diff.lua` replaces specific line ranges when reverting a hunk:

```lua
-- Replace lines [new_start-1, new_start-1+new_count) with old_lines
vim.api.nvim_buf_set_lines(
    bufnr,
    h.new_start - 1,                   -- 0-indexed start
    h.new_start - 1 + h.new_count,     -- 0-indexed exclusive end
    false,
    old_lines                           -- replacement lines
)
```

This is how you surgically replace a range of lines. The API uses 0-based
indexing with an exclusive end, so lines 5-7 (1-indexed) would be
`nvim_buf_set_lines(buf, 4, 7, false, new_lines)`.

### Forcing Buffer Reload

When an external tool modifies a file, you need to tell Neovim to re-read it:

```lua
vim.o.autoread = true  -- Enable auto-read globally (in setup)

-- For a specific buffer:
vim.api.nvim_buf_call(bufnr, function()
    vim.cmd("checktime")
end)
```

`nvim_buf_call` executes a function in the context of a specific buffer, which
is necessary because `checktime` operates on the current buffer.

---

## 8. Extmarks and Highlights

Extmarks are the modern way to add visual decorations to buffers. They are
more flexible and reliable than the older `matchadd` and `buf_add_highlight`
APIs.

### Creating Namespaces

A namespace isolates your extmarks from other plugins. Always create one:

```lua
local ns = vim.api.nvim_create_namespace("claude_diff")
local sign_ns = vim.api.nvim_create_namespace("claude_diff_signs")
local flash_ns = vim.api.nvim_create_namespace("claude_flash")
```

You can have multiple namespaces for different purposes (inline highlights,
sign column, temporary effects). This lets you clear one category without
affecting the others.

### Defining Highlight Groups

`highlights.lua` defines all highlight groups in one place, linked to
built-in semantic groups:

```lua
local defined = false

function M.setup()
    if defined then return end
    defined = true

    local hl = function(name, opts)
        opts.default = true
        vim.api.nvim_set_hl(0, name, opts)
    end

    -- Sidebar highlights: link to semantic groups
    hl("ClaudeHeader",   { link = "Title" })
    hl("ClaudeActive",   { link = "DiagnosticOk" })
    hl("ClaudeInactive", { link = "Comment" })
    hl("ClaudeLive",     { link = "DiagnosticWarn" })
    hl("ClaudeSession",  { link = "DiagnosticInfo" })
    hl("ClaudeDir",      { link = "Directory" })
    hl("ClaudeFile",     { link = "Normal" })
    hl("ClaudeFileCurrent", { bold = true, underline = true, link = "Normal" })

    -- Diff highlights: link to built-in diff groups
    hl("ClaudeDiffAdd",        { link = "DiffAdd" })
    hl("ClaudeDiffChange",     { link = "DiffChange" })
    hl("ClaudeDiffDelete",     { link = "DiffDelete" })
    hl("ClaudeDiffAddSign",    { link = "Added" })
    hl("ClaudeDiffChangeSign", { link = "Changed" })
    hl("ClaudeDiffDeleteSign", { link = "Removed" })
end
```

Key design decisions:

- **`default = true`** means the highlight is only set if the user has not
  already defined it. Users can override any group in their colorscheme.
- **`link`** ties to built-in semantic groups (`Title`, `DiffAdd`, `Comment`,
  etc.) that every colorscheme defines. Your plugin adapts to any theme
  automatically.
- **Idempotent** -- the `defined` flag prevents re-registration.

`vim.api.nvim_set_hl(0, name, opts)` sets a global highlight. The `0`
means the global namespace (as opposed to a per-window namespace).

### Setting Extmarks: Line Highlights

Highlight an entire line (used for diff additions):

```lua
pcall(vim.api.nvim_buf_set_extmark, bufnr, ns, line - 1, 0, {
    line_hl_group = "ClaudeDiffAdd",
    sign_text = "┃",
    sign_hl_group = "ClaudeDiffAddSign",
})
```

- `line - 1` because extmark lines are 0-indexed
- `line_hl_group` highlights the entire line background
- `sign_text` puts a character in the sign column
- `sign_hl_group` colors the sign character

### Setting Extmarks: Virtual Text (Deleted Lines)

Show deleted lines as virtual text above or below a real line:

```lua
local virt_lines = {}
for i = old_start, old_start + old_count - 1 do
    if before_lines[i] then
        virt_lines[#virt_lines + 1] = {
            { "  - ", "ClaudeDiffDeleteNr" },
            { before_lines[i], "ClaudeDiffDelete" },
        }
    end
end

pcall(vim.api.nvim_buf_set_extmark, bufnr, ns, math.max(0, new_start - 1), 0, {
    virt_lines = virt_lines,
    virt_lines_above = (new_start > 0),
    sign_text = "▁",
    sign_hl_group = "ClaudeDiffDeleteSign",
})
```

Each entry in `virt_lines` is a list of `{ text, highlight_group }` tuples.
This lets you color different parts of a virtual line differently (the `- `
prefix in red, the deleted text in a different shade).

### Setting Extmarks: Sign Column Only

For lightweight indicators (not full inline diff), use sign-only extmarks:

```lua
function M.apply_signs(bufnr, filepath)
    highlights.setup()

    local hunks = compute_hunks(filepath, bufnr)
    vim.api.nvim_buf_clear_namespace(bufnr, sign_ns, 0, -1)
    if not hunks or #hunks == 0 then return end

    for _, hunk in ipairs(hunks) do
        local old_count, new_start, new_count = hunk[2], hunk[3], hunk[4]
        if old_count == 0 and new_count > 0 then
            -- Added lines: green bar
            pcall(vim.api.nvim_buf_set_extmark, bufnr, sign_ns, new_start - 1, 0, {
                end_row = new_start + new_count - 2,
                sign_text = "┃",
                sign_hl_group = "ClaudeDiffAddSign",
                number_hl_group = "ClaudeDiffAddSign",
                priority = 15,
            })
        elseif new_count == 0 then
            -- Deleted lines: bottom bar marker
            pcall(vim.api.nvim_buf_set_extmark, bufnr, sign_ns,
                math.max(0, new_start - 1), 0, {
                sign_text = "▁",
                sign_hl_group = "ClaudeDiffDeleteSign",
                priority = 15,
            })
        else
            -- Changed lines: yellow bar
            pcall(vim.api.nvim_buf_set_extmark, bufnr, sign_ns, new_start - 1, 0, {
                end_row = new_start + new_count - 2,
                sign_text = "┃",
                sign_hl_group = "ClaudeDiffChangeSign",
                number_hl_group = "ClaudeDiffChangeSign",
                priority = 15,
            })
        end
    end
end
```

- **`end_row`** spans the extmark across multiple lines.
- **`number_hl_group`** also colors the line number column.
- **`priority`** controls which extmark wins when multiple overlap. Higher
  numbers win. Choose a priority that makes sense relative to other plugins
  (gitsigns typically uses 10, so 15 puts your signs slightly above).

### Setting Extmarks: Sidebar Text Highlighting

The sidebar uses extmarks for per-segment highlighting (indicator, icon,
filename, stats all have different colors on the same line):

```lua
vim.api.nvim_buf_clear_namespace(sidebar_buf, ns, 0, -1)
for _, h in ipairs(hls) do
    pcall(vim.api.nvim_buf_set_extmark, sidebar_buf, ns, h[1], h[3] or 0, {
        end_col = (h[4] and h[4] >= 0) and h[4] or nil,
        hl_group = h[2],
        hl_eol = (h[4] or -1) == -1,
    })
end
```

- **`h[3]` and `h[4]`** are byte-offset start and end columns within the line.
- **`hl_eol = true`** extends the highlight to the end of the line.
- When `end_col` is nil, the highlight covers from `col` to end-of-line.

### Temporary Flash Effects

`diff.lua` creates a brief highlight when jumping to a hunk:

```lua
local function flash_line(bufnr, line)
    pcall(vim.api.nvim_buf_set_extmark, bufnr, flash_ns, line - 1, 0, {
        line_hl_group = "IncSearch",
        priority = 100,
    })
    vim.defer_fn(function()
        pcall(vim.api.nvim_buf_clear_namespace, bufnr, flash_ns, 0, -1)
    end, 300)
end
```

`vim.defer_fn(fn, ms)` schedules a function to run after a delay. Combined
with a dedicated namespace, this creates a flash-and-fade effect.

### Clearing Extmarks

Always clear your namespace before re-rendering or when cleaning up:

```lua
vim.api.nvim_buf_clear_namespace(bufnr, ns, 0, -1)
-- Clears all extmarks in namespace `ns` from line 0 to the end
```

---

## 9. Windows and Splits

### Creating a Sidebar Window

`sidebar.lua` creates a left-side vertical split:

```lua
function M.open()
    if is_open() then
        M.render()
        return
    end

    highlights.setup()

    -- Create the buffer first
    sidebar_buf = vim.api.nvim_create_buf(false, true)
    vim.bo[sidebar_buf].buftype = "nofile"
    vim.bo[sidebar_buf].bufhidden = "wipe"
    vim.bo[sidebar_buf].swapfile = false
    vim.bo[sidebar_buf].filetype = "claude-sidebar"

    -- Create a vertical split on the left
    vim.cmd("topleft vsplit")
    sidebar_win = vim.api.nvim_get_current_win()
    vim.api.nvim_win_set_buf(sidebar_win, sidebar_buf)
    vim.api.nvim_win_set_width(sidebar_win, get_width())

    -- Configure window options
    local wo = vim.wo[sidebar_win]
    wo.number = false
    wo.relativenumber = false
    wo.signcolumn = "no"
    wo.cursorcolumn = false
    wo.foldcolumn = "0"
    wo.wrap = false
    wo.winfixwidth = true    -- Prevent resize on Ctrl-W =
    wo.cursorline = true     -- Highlight the line under cursor
    wo.statusline = " "      -- Minimal statusline

    -- Set buffer-local keymaps (see Keymaps section)
    -- ...

    M.render()
    vim.cmd("wincmd p")  -- Return focus to the previous window
end
```

The sequence is:

1. Create a buffer with `nvim_create_buf`.
2. Create a window with `vim.cmd("topleft vsplit")`.
3. Assign the buffer to the window with `nvim_win_set_buf`.
4. Configure window options via `vim.wo[win]`.
5. Return focus to the main editor window with `wincmd p`.

`winfixwidth = true` is essential for sidebars -- it prevents `Ctrl-W =` from
equalizing your sidebar width with other windows.

### Checking if a Window is Open

```lua
local function is_open()
    return sidebar_win and vim.api.nvim_win_is_valid(sidebar_win)
        and sidebar_buf and vim.api.nvim_buf_is_valid(sidebar_buf)
end
```

Always validate both the window and buffer handles. A window can be closed
by the user at any time, making the handle stale.

### Closing a Window

```lua
function M.close()
    if sidebar_win and vim.api.nvim_win_is_valid(sidebar_win) then
        vim.api.nvim_win_close(sidebar_win, true)
    end
    sidebar_win = nil
    sidebar_buf = nil  -- Buffer auto-wiped because bufhidden = "wipe"
end
```

The second argument to `nvim_win_close` is `force` -- if `true`, it closes the
window even if the buffer has unsaved changes.

### Toggle Pattern

```lua
function M.toggle()
    if is_open() then M.close() else M.open() end
end
```

Simple but important for UX. Map this to a keymap and the user can press the
same key to open and close.

### Floating Windows

`sidebar.lua` creates a floating window for the help popup:

```lua
local function show_help()
    local help = {
        "...",  -- lines of help text
    }
    local buf = vim.api.nvim_create_buf(false, true)
    vim.api.nvim_buf_set_lines(buf, 0, -1, false, help)
    vim.bo[buf].modifiable = false
    vim.bo[buf].bufhidden = "wipe"

    local win = vim.api.nvim_open_win(buf, true, {
        relative = "editor",
        width = 34,
        height = #help,
        row = math.floor((vim.o.lines - #help) / 2),
        col = math.floor((vim.o.columns - 34) / 2),
        style = "minimal",
        border = "none",
    })

    -- Close on q, g?, or Escape
    vim.keymap.set("n", "q", function()
        vim.api.nvim_win_close(win, true)
    end, { buffer = buf, nowait = true })
    vim.keymap.set("n", "<Esc>", function()
        vim.api.nvim_win_close(win, true)
    end, { buffer = buf, nowait = true })
end
```

`nvim_open_win` options:

- **`relative = "editor"`** -- position relative to the entire editor. Other
  options: `"win"` (relative to a window), `"cursor"` (relative to cursor).
- **`row`, `col`** -- top-left corner position.
- **`style = "minimal"`** -- no line numbers, no statusline, no sign column.
- **`border`** -- `"none"`, `"single"`, `"double"`, `"rounded"`, `"shadow"`,
  or a custom table.

### Navigating Between Windows

When the user selects a file in the sidebar, open it in the main window:

```lua
local function open_with_diff()
    local f = file_at_cursor()
    if not f then
        vim.notify("No file on this line", vim.log.levels.INFO)
        return
    end
    -- If only one window, create a split
    if vim.fn.winnr("$") <= 1 then
        vim.cmd("wincmd v")
    end
    vim.cmd("wincmd p")  -- Move to previous (non-sidebar) window
    vim.cmd("edit " .. vim.fn.fnameescape(f.abs))
    require("cc-watcher.diff").show(f.abs, { jump = true })
end
```

`vim.fn.fnameescape()` is critical when building command strings with file
paths -- it escapes spaces and special characters.

---

## 10. File I/O with libuv

Neovim embeds libuv, accessible via `vim.uv` (or `vim.loop` in older
versions). This gives you low-level, efficient file system operations.

### Reading Files

`util.lua` provides a helper that reads an entire file:

```lua
M.FILE_MODE = 384  -- octal 0600: rw for owner only

function M.read_file(filepath)
    local fd = vim.uv.fs_open(filepath, "r", M.FILE_MODE)
    if not fd then return nil end
    local stat = vim.uv.fs_fstat(fd)
    if not stat or stat.size == 0 then
        vim.uv.fs_close(fd)
        return stat and "" or nil
    end
    local data = vim.uv.fs_read(fd, stat.size, 0)
    vim.uv.fs_close(fd)
    return data
end
```

The sequence: `fs_open` -> `fs_fstat` (get size) -> `fs_read` (read all
bytes) -> `fs_close`. Always close the file descriptor, even on error paths.

`fs_fstat` on an open fd is safer than `fs_stat` on a path because there is
no race condition (TOCTOU) between stat and read.

### Watching Files with fs_event

`watcher.lua` uses libuv's `fs_event` to monitor files for changes:

```lua
local function watch_file(filepath)
    if file_watchers[filepath] or M.should_ignore(filepath) then return end

    local handle = vim.uv.new_fs_event()
    if not handle then return end

    handle:start(filepath, {}, vim.schedule_wrap(function(err, _, events)
        if err then
            unwatch_file(filepath)
            return
        end
        if events and events.rename then
            unwatch_file(filepath)
            return
        end
        if not events or not events.change then return end

        -- File changed -- compare against snapshot
        local snap = snapshots.get(filepath)
        if not snap then return end

        local fd = vim.uv.fs_open(filepath, "r", 438)
        if not fd then return end
        local stat = vim.uv.fs_fstat(fd)
        if not stat then vim.uv.fs_close(fd); return end
        if stat.mtime.sec == snap.mtime then vim.uv.fs_close(fd); return end

        local data = vim.uv.fs_read(fd, stat.size, 0)
        vim.uv.fs_close(fd)
        if not data then return end

        if data ~= snap.raw then
            M.mark_changed(filepath)
            -- Trigger buffer reload
            for _, bufnr in ipairs(vim.api.nvim_list_bufs()) do
                if vim.api.nvim_buf_is_loaded(bufnr)
                    and vim.api.nvim_buf_get_name(bufnr) == filepath then
                    vim.api.nvim_buf_call(bufnr, function()
                        vim.cmd("checktime")
                    end)
                    break
                end
            end
        end
    end))

    file_watchers[filepath] = handle
end
```

Key details:

- **`vim.uv.new_fs_event()`** creates a new file system event handle.
- **`handle:start(path, flags, callback)`** begins watching.
- The callback receives `err`, `filename`, and `events` (`{ change = true }` or
  `{ rename = true }`).
- **Always wrap the callback with `vim.schedule_wrap`** because libuv callbacks
  run outside the Neovim main loop. Without this, calling Neovim APIs inside
  the callback would crash.
- **`events.rename`** fires when a file is deleted, renamed, or replaced. Clean
  up the watcher when this happens.

### Cleaning Up Watchers

Always stop and close libuv handles when done:

```lua
local function unwatch_file(filepath)
    local h = file_watchers[filepath]
    if h then
        h:stop()
        h:close()
        file_watchers[filepath] = nil
    end
end
```

Failing to close handles causes resource leaks. This cleanup is also
triggered from autocommands (`BufDelete`, `BufWipeout`).

### Re-watching After File Replacement

Some tools (like Claude Code) replace files by writing to a temp file and
renaming. This triggers `events.rename`, which kills the watcher.
`session.lua` handles this by re-watching after a delay:

```lua
jsonl_watcher_handle:start(path, {}, vim.schedule_wrap(function(err, _, events)
    if err then return end
    if events and events.rename then
        vim.defer_fn(function() M.watch_jsonl(cwd) end, 200)
        return
    end
    if events and events.change then
        for _, cb in ipairs(jsonl_change_callbacks) do
            pcall(cb, path)
        end
    end
end))
```

The 200ms delay gives the filesystem time to settle after the rename.

### Scanning Directories

`session.lua` scans directories using `fs_scandir`:

```lua
local handle = vim.uv.fs_scandir(project_dir)
if not handle then return nil end

local best_path, best_mtime = nil, 0
while true do
    local name, typ = vim.uv.fs_scandir_next(handle)
    if not name then break end
    if typ == "file" and name:match("%.jsonl$") then
        local fpath = project_dir .. "/" .. name
        local stat = vim.uv.fs_stat(fpath)
        if stat and stat.mtime.sec > best_mtime then
            best_path = fpath
            best_mtime = stat.mtime.sec
        end
    end
end
```

`fs_scandir` returns a handle, and `fs_scandir_next` iterates entries.
Each entry has a name and type (`"file"`, `"directory"`, etc.).

### Timers

`sidebar.lua` uses libuv timers for debouncing:

```lua
-- Create reusable timers (module-level, not per-call)
local debounce_timer = vim.uv.new_timer()
local jsonl_debounce = vim.uv.new_timer()

-- Usage in a callback:
watcher.on_change(function(filepath, rel)
    pending_changes[#pending_changes + 1] = rel
    debounce_timer:stop()
    debounce_timer:start(500, 0, vim.schedule_wrap(flush_notifications))
end)
```

The pattern: `timer:stop()` cancels any pending firing, then
`timer:start(timeout_ms, repeat_ms, callback)` starts a new countdown.
With `repeat_ms = 0`, it fires once. This is a standard debounce: rapid
changes restart the timer, and the callback only fires 500ms after the
last change.

Creating timers at the module level and reusing them avoids handle leaks.
If you created a new timer in every callback, you would leak handles.

### Checking if a Process is Running

`session.lua` checks if a Claude Code process is still alive:

```lua
local pid = tonumber(sess.pid)
if pid and pid > 0 and vim.uv.kill(pid, 0) == 0 then
    -- Process is running
end
```

`vim.uv.kill(pid, 0)` sends signal 0, which does not kill the process but
checks if it exists. Returns 0 on success (process exists).

---

## 11. Shell Commands

### `vim.fn.systemlist` and `vim.v.shell_error`

The simplest way to run a shell command and capture output:

```lua
local git_rel, git_dir = util.git_relpath(filepath)
local cmd = "git -C " .. vim.fn.shellescape(git_dir)
    .. " show HEAD:" .. vim.fn.shellescape(git_rel) .. " 2>/dev/null"
local lines = vim.fn.systemlist(cmd)
if vim.v.shell_error == 0 and #lines > 0 then
    return lines
end
```

- **`vim.fn.systemlist(cmd)`** runs the command and returns output as a
  list of lines (strings).
- **`vim.v.shell_error`** is the exit code of the last shell command.
  Check it immediately after `systemlist()`.
- **`vim.fn.shellescape(str)`** wraps a string in quotes and escapes
  special characters. Always use this when interpolating user-provided
  or file-derived strings into shell commands.
- **`2>/dev/null`** suppresses stderr in the shell command itself.

### Getting Git-Relative Paths

`util.lua` combines shell commands with path manipulation:

```lua
function M.git_relpath(filepath)
    local dir = vim.fn.fnamemodify(filepath, ":h")
    local toplevel = vim.fn.systemlist(
        "git -C " .. vim.fn.shellescape(dir)
        .. " rev-parse --show-toplevel 2>/dev/null"
    )[1]
    if vim.v.shell_error ~= 0 or not toplevel or toplevel == "" then
        return nil, nil
    end
    if filepath:sub(1, #toplevel + 1) == toplevel .. "/" then
        return filepath:sub(#toplevel + 2), dir
    end
    return nil, nil
end
```

`vim.fn.fnamemodify(path, ":h")` extracts the directory part of a path
(head). Other useful modifiers:

- `:t` -- tail (filename only)
- `:r` -- root (remove extension)
- `:e` -- extension only
- `:p` -- make absolute
- `:~` -- relative to home directory

### Computing Diffs

Neovim has a built-in `vim.diff` function:

```lua
-- Index-based hunks (for programmatic use):
local hunks = vim.diff(old_text, new_text, {
    result_type = "indices",
    algorithm = "histogram",
})
-- Returns: { { old_start, old_count, new_start, new_count }, ... }

-- Unified diff string (for display):
local unified = vim.diff(old_text, new_text, {
    result_type = "unified",
    ctxlen = 3,  -- context lines
})
```

This is a pure-Lua diff implementation using xdiff, the same library git uses.
No subprocess needed.

---

## 12. Integrating with Other Plugins

### trouble.nvim Source

trouble.nvim v3 auto-discovers sources from `lua/trouble/sources/*.lua`. To
register your plugin as a trouble source, create a proxy file at that path:

```
lua/trouble/sources/claude.lua
```

With contents:

```lua
-- Proxy: trouble.nvim auto-discovers sources from lua/trouble/sources/*.lua
-- This delegates to the actual implementation in cc-watcher.
return require("cc-watcher.trouble")
```

The actual implementation in `cc-watcher/trouble.lua` exports a `config`
table (with mode definitions) and a `get` function:

```lua
local Item = require("trouble.item")
local util = require("cc-watcher.util")

local M = {}

M.config = {
    modes = {
        claude = {
            desc = "Claude Code Changes",
            source = "claude",
            groups = {
                { "filename", format = "{file_icon} {filename} {count}" },
            },
            sort = { "severity", "filename", "pos" },
            format = "{severity_icon} {text:ts} {pos}",
        },
    },
}

function M.get(cb, ctx)
    util.collect_files(function(files)
        vim.schedule(function()
            local items = {}
            for _, f in ipairs(files) do
                local old_text = util.get_old_text(f.abs)
                local new_text = util.read_file(f.abs)
                if new_text then
                    local hunks = util.compute_hunks(old_text, new_text)
                    if hunks then
                        for _, h in ipairs(hunks) do
                            items[#items + 1] = Item.new({
                                pos = { row, 0 },
                                end_pos = { row + math.max(new_count, 1) - 1, 0 },
                                text = hunk_description(old_count, new_count),
                                severity = hunk_severity(old_count, new_count),
                                filename = f.abs,
                                source = "claude",
                            })
                        end
                    end
                end
            end
            Item.add_id(items, { "severity" })
            Item.add_text(items, { mode = "full" })
            cb(items)
        end)
    end)
end

return M
```

The trouble source API expects:

- `M.config.modes` -- defines a named mode with display format, sorting, grouping.
- `M.get(cb, ctx)` -- an async function that collects items and passes them to `cb`.
- Each item is created with `Item.new({ pos, text, severity, filename, ... })`.

### snacks.nvim Picker

snacks.nvim provides a generic picker API. Here is how cc-watcher creates a
file picker with custom formatting and diff preview:

```lua
function M.changed_files()
    local ok, Snacks = pcall(require, "snacks")
    if not ok then
        vim.notify("snacks.nvim is required for this feature", vim.log.levels.ERROR)
        return
    end

    util.collect_files(function(files, cwd)
        vim.schedule(function()
            local items = {}
            for _, f in ipairs(files) do
                items[#items + 1] = {
                    text = f.rel,
                    file = f.abs,
                    indicator = f.live and "● " or "○ ",
                    stats = stats_string,
                    rel = f.rel,
                    live = f.live,
                }
            end

            Snacks.picker({
                title = "Claude Changed Files",
                items = items,
                format = function(item)
                    local ret = {}
                    ret[#ret + 1] = { item.indicator,
                        item.live and "ClaudeLive" or "ClaudeSession" }
                    ret[#ret + 1] = { item.rel }
                    if item.stats ~= "" then
                        ret[#ret + 1] = { item.stats, "ClaudeStats" }
                    end
                    return ret
                end,
                preview = function(ctx)
                    -- Compute diff and render in preview buffer
                    -- (see snacks.lua for full implementation)
                end,
                confirm = function(picker, item)
                    picker:close()
                    vim.cmd("edit " .. vim.fn.fnameescape(item.file))
                    require("cc-watcher.diff").show(item.file, { jump = true })
                end,
            })
        end)
    end)
end
```

The snacks picker expects:

- `items` -- a list of tables with at least a `text` field for filtering.
- `format` -- a function returning `{ { text, hl_group }, ... }` segments.
- `preview` -- a function receiving a context with `ctx.preview` for rendering.
- `confirm` -- called when the user selects an item.

### Lualine Statusline Component

`init.lua` exports a statusline function that any statusline plugin can use:

```lua
function M.statusline()
    local ok, watcher = pcall(require, "cc-watcher.watcher")
    if not ok then return "" end
    local n = vim.tbl_count(watcher.get_changed_files())
    if n == 0 then return "" end
    return "some-icon " .. n
end
```

Users add it to their lualine config:

```lua
-- In lualine setup:
sections = {
    lualine_x = {
        { require("cc-watcher").statusline },
    },
}
```

The function must return a string, return `""` when there is nothing to show,
and be cheap to call (it runs on every statusline redraw).

### Gating Integrations Behind Config Flags

cc-watcher requires explicit opt-in for each integration:

```lua
vim.api.nvim_create_user_command("ClaudeTrouble", function()
    ensure()
    local cfg = require("cc-watcher").config
    if not cfg.integrations.trouble then
        vim.notify(
            "cc-watcher: trouble integration is disabled. "
            .. "Enable it with integrations.trouble = true",
            vim.log.levels.WARN
        )
        return
    end
    local ok, trouble = pcall(require, "trouble")
    if not ok then
        vim.notify("cc-watcher: trouble.nvim not found", vim.log.levels.ERROR)
        return
    end
    trouble.open({ mode = "claude" })
end, { desc = "Trouble: Claude Code changes" })
```

The double gate (config flag + pcall require) is defensive:

1. The config flag lets users avoid even attempting to load an optional
   dependency.
2. The `pcall(require, ...)` catches the case where the integration is
   enabled but the dependency is not installed.

### The `pcall(require, ...)` Pattern

This is the standard Neovim pattern for optional dependencies:

```lua
local ok, module = pcall(require, "some-optional-plugin")
if not ok then
    -- Plugin not installed; degrade gracefully
    return
end
-- Plugin is available; use `module`
```

cc-watcher uses this everywhere:

- `pcall(require, "which-key")` for which-key integration
- `pcall(require, "snacks")` in snacks.lua
- `pcall(require, "trouble")` in command registration
- `pcall(require, "nvim-web-devicons")` for file icons in the sidebar

---

## 13. Debugging

### `vim.notify`

The primary way to communicate with the user:

```lua
vim.notify("No snapshot or git history for this file", vim.log.levels.WARN)
vim.notify(#hunks .. " hunk(s)  ]c/[c nav  cr revert", vim.log.levels.INFO)
vim.notify("cc-watcher: trouble.nvim not found", vim.log.levels.ERROR)
```

Log levels: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`. If the user has a
notification plugin (like nvim-notify or fidget.nvim), these show as nice
popups. Otherwise they appear in the message area.

For development, you can add a title:

```lua
vim.notify("Changed file: " .. rel, vim.log.levels.INFO, { title = "Claude" })
```

### `vim.inspect`

Convert any Lua value to a readable string:

```lua
print(vim.inspect(some_table))
-- Or in a notify:
vim.notify(vim.inspect(M.config), vim.log.levels.DEBUG)
```

This is invaluable for debugging nested tables, especially config tables
and event args.

### The `:lua` Command

Run arbitrary Lua from command mode:

```
:lua print(vim.inspect(require("cc-watcher").config))
:lua print(vim.inspect(require("cc-watcher.watcher").get_changed_files()))
:lua =require("cc-watcher.session").find_latest_jsonl(vim.fn.getcwd())
```

The `:lua =expr` shorthand (note the `=`) pretty-prints the result.

### `:messages`

View the message history, including notifications and errors:

```
:messages
```

This captures output from `print()`, `vim.notify()`, and error messages.
Clear with `:messages clear`.

### Reloading Modules During Development

Lua's `require` cache means changes to source files are not picked up until
you explicitly reload:

```lua
-- Clear cached module and re-require:
package.loaded["cc-watcher.sidebar"] = nil
require("cc-watcher.sidebar")
```

Or clear the entire plugin:

```lua
for key in pairs(package.loaded) do
    if key:match("^cc%-watcher") then
        package.loaded[key] = nil
    end
end
```

Be aware that this does not undo side effects (autocommands, keymaps,
extmarks). For a clean reload, restart Neovim.

### Common Pitfalls

**1. Calling Neovim APIs from libuv callbacks:**

```lua
-- WRONG: will crash
handle:start(path, {}, function(err, _, events)
    vim.api.nvim_buf_set_lines(buf, 0, -1, false, {})  -- crash!
end)

-- CORRECT: schedule onto main loop
handle:start(path, {}, vim.schedule_wrap(function(err, _, events)
    vim.api.nvim_buf_set_lines(buf, 0, -1, false, {})  -- safe
end))
```

**2. Not checking buffer/window validity:**

```lua
-- WRONG: window might have been closed
vim.api.nvim_win_set_cursor(sidebar_win, { 1, 0 })

-- CORRECT: check first
if sidebar_win and vim.api.nvim_win_is_valid(sidebar_win) then
    pcall(vim.api.nvim_win_set_cursor, sidebar_win, { 1, 0 })
end
```

**3. Forgetting to close libuv handles:**

```lua
-- WRONG: handle leaks on error
local fd = vim.uv.fs_open(filepath, "r", 438)
local stat = vim.uv.fs_fstat(fd)
if not stat then return end  -- fd is leaked!

-- CORRECT: always close
local fd = vim.uv.fs_open(filepath, "r", 438)
local stat = vim.uv.fs_fstat(fd)
if not stat then vim.uv.fs_close(fd); return end
local data = vim.uv.fs_read(fd, stat.size, 0)
vim.uv.fs_close(fd)
```

**4. Off-by-one errors with API indexing:**

Neovim APIs use 0-based line indexing. Lua tables use 1-based indexing.
Cursor positions use 1-based `{ row, col }`. Extmarks use 0-based
`{ line, col }`. This is the single most common source of bugs.

```lua
-- Cursor row is 1-based:
local row = vim.api.nvim_win_get_cursor(0)[1]  -- 1-indexed

-- Extmark line is 0-based:
vim.api.nvim_buf_set_extmark(buf, ns, row - 1, 0, { ... })

-- buf_set_lines uses 0-based indexing:
vim.api.nvim_buf_set_lines(buf, 0, -1, false, lines)  -- 0 = first line
```

**5. Forgetting `pcall` around optional operations:**

```lua
-- WRONG: crashes if keymap doesn't exist
vim.keymap.del("n", "]c", { buffer = bufnr })

-- CORRECT: graceful failure
pcall(vim.keymap.del, "n", "]c", { buffer = bufnr })
```

---

## 14. Testing

### Setting Up mini.test

cc-watcher uses mini.test from mini.nvim for its test suite. The setup
requires three files.

**`tests/minimal_init.lua`** -- a minimal Neovim config that loads only
what the tests need:

```lua
-- Minimal nvim config for running tests headless
vim.opt.rtp:prepend(".")
vim.opt.rtp:prepend("deps/mini.nvim")

vim.o.swapfile = false
vim.o.shadafile = "NONE"

require("mini.test").setup()
```

`vim.opt.rtp:prepend(".")` adds the plugin root to the runtime path so that
`require("cc-watcher")` finds your modules. `"deps/mini.nvim"` is where the
test dependency lives.

**`Makefile`** -- the test runner:

```makefile
TESTS_DIR := tests
MINI_NVIM := deps/mini.nvim

.PHONY: deps test test-file clean

deps:
	@mkdir -p deps
	@[ -d $(MINI_NVIM) ] || git clone --depth 1 \
		https://github.com/echasnovski/mini.nvim $(MINI_NVIM)

test: deps
	nvim --headless -u tests/minimal_init.lua \
		-c "lua MiniTest.run()"

test-file: deps
	nvim --headless -u tests/minimal_init.lua \
		-c "lua MiniTest.run_file('$(FILE)')"

clean:
	rm -rf deps
```

Run all tests with `make test`, or a single file with
`make test-file FILE=tests/test_snapshots.lua`.

### Writing Test Files

Test files follow the mini.test structure:

```lua
local T = MiniTest.new_set()

T["module_name"] = MiniTest.new_set()

T["module_name"]["descriptive test name"] = function()
    -- Arrange
    local cc = require("cc-watcher")

    -- Act
    cc.setup({ sidebar_width = 42 })

    -- Assert
    MiniTest.expect.equality(cc.config.sidebar_width, 42)
end

return T
```

Key patterns from the cc-watcher test suite:

**Testing config merging (`test_init.lua`):**

```lua
T["init"]["setup() deep merges nested tables"] = function()
    local cc = require("cc-watcher")

    cc.setup({
        keys = { toggle_sidebar = "<leader>cc" },
        integrations = { snacks = true },
    })

    -- Overridden
    MiniTest.expect.equality(cc.config.keys.toggle_sidebar, "<leader>cc")
    MiniTest.expect.equality(cc.config.integrations.snacks, true)

    -- Defaults preserved for siblings
    MiniTest.expect.equality(cc.config.keys.toggle_diff, "<leader>cd")
    MiniTest.expect.equality(cc.config.integrations.fzf_lua, false)
end
```

**Testing with temp files and hooks (`test_snapshots.lua`):**

```lua
local snapshots = require("cc-watcher.snapshots")

T["snapshots"] = MiniTest.new_set({
    hooks = {
        pre_case = function() snapshots.clear() end,
        post_case = function() snapshots.clear() end,
    },
})

T["snapshots"]["take() captures file contents"] = function()
    local tmp = vim.fn.tempname()
    vim.fn.writefile({ "hello", "world" }, tmp)

    snapshots.take(tmp)

    MiniTest.expect.equality(snapshots.has(tmp), true)
    local snap = snapshots.get(tmp)
    MiniTest.expect.equality(snap.lines[1], "hello")
    MiniTest.expect.equality(snap.lines[2], "world")
    MiniTest.expect.equality(type(snap.raw), "string")
    MiniTest.expect.equality(type(snap.mtime), "number")

    vim.fn.delete(tmp)
end
```

Hooks run before and after each test case. Use them to reset state so tests
do not interfere with each other.

**Testing edge cases:**

```lua
T["snapshots"]["LRU evicts oldest when over capacity"] = function()
    local files = {}
    for i = 1, 102 do
        local tmp = vim.fn.tempname()
        vim.fn.writefile({ "file" .. i }, tmp)
        files[i] = tmp
        snapshots.take(tmp)
    end

    -- Should have evicted the first 2 (max capacity is 100)
    MiniTest.expect.equality(snapshots.count(), 100)
    MiniTest.expect.equality(snapshots.has(files[1]), false)
    MiniTest.expect.equality(snapshots.has(files[2]), false)
    MiniTest.expect.equality(snapshots.has(files[3]), true)
    MiniTest.expect.equality(snapshots.has(files[102]), true)

    for _, f in ipairs(files) do vim.fn.delete(f) end
end
```

### Providing `_reset()` Methods for Testing

Several cc-watcher modules expose `_reset()` functions specifically for
testing:

```lua
-- In watcher.lua:
function M._reset()
    changed_files = {}
    on_change_callbacks = {}
end

-- In session.lua:
function M._reset()
    tail.jsonl_path = nil
    tail.offset = 0
    tail.seen = {}
    tail.files = {}
    jsonl_dir_cache.cwd = nil
    -- ...
end
```

This lets tests reset internal state without restarting Neovim. Prefix with
`_` to signal these are internal/testing-only APIs.

### Assertions

mini.test provides:

```lua
MiniTest.expect.equality(actual, expected)  -- Deep equality
MiniTest.expect.no_equality(actual, expected)
MiniTest.expect.error(fn)  -- Expect fn to throw
```

For more complex assertions, just use Lua:

```lua
local result = some_function()
assert(type(result) == "table", "expected table, got " .. type(result))
assert(#result > 0, "expected non-empty result")
```

---

## 15. Publishing

### Repository Structure

A publishable Neovim plugin repository should contain:

```
my-plugin.nvim/
  lua/my-plugin/        -- Your Lua source
  plugin/my-plugin.lua  -- Entry point
  doc/my-plugin.txt     -- Vimdoc help file
  tests/                -- Test suite
  Makefile              -- Test runner
  README.md             -- GitHub-facing documentation
  LICENSE               -- License file (MIT is common)
```

### README Structure

A good README for a Neovim plugin typically includes:

1. **Title and one-line description** -- what it does in plain language.
2. **Screenshot or demo** -- a GIF or image showing the plugin in action.
3. **Features** -- bullet list of capabilities.
4. **Requirements** -- Neovim version, dependencies.
5. **Installation** -- lazy.nvim, packer, or manual instructions.
6. **Configuration** -- the defaults table and explanation of each option.
7. **Usage** -- commands, keymaps, API functions.
8. **Integrations** -- how it works with other plugins.
9. **License**.

### Vimdoc

The `doc/` directory contains Neovim help files in vimdoc format. These are
plain text with a specific markup:

```
*my-plugin.txt*    Description of my plugin

CONTENTS                                            *my-plugin-contents*

1. Introduction .......................... |my-plugin-introduction|
2. Configuration ......................... |my-plugin-configuration|

==============================================================================
1. INTRODUCTION                                     *my-plugin-introduction*

My plugin does amazing things.

==============================================================================
2. CONFIGURATION                                    *my-plugin-configuration*

                                                    *my-plugin.setup()*
setup({opts})
    Configure the plugin.

    Parameters: ~
        {opts}  (table)  Configuration table
            - {sidebar_width}  (number)  Width of sidebar (default: 36)
            - {keys}           (table)   Keymap configuration
```

After writing or updating the help file, regenerate tags:

```
:helptags doc/
```

This creates a `tags` file that enables `:help my-plugin` to jump to the
right location.

### lazy.nvim Compatibility

For lazy.nvim users, your plugin should work with this minimal spec:

```lua
{
    "your-github-username/my-plugin.nvim",
    opts = {},  -- Calls require("my-plugin").setup(opts)
}
```

This works because lazy.nvim:

1. Clones the repo.
2. Adds it to the runtime path.
3. Runs `plugin/my-plugin.lua` (registering commands).
4. Calls `require("my-plugin").setup(opts)` with the user's opts.

For deferred loading, export trigger hints like cc-watcher does:

```lua
-- In init.lua:
M.lazy = {
    cmd = { "MySidebarToggle", "MyDiffShow" },
    keys = {
        { "<leader>ms", function() ... end, desc = "My sidebar" },
    },
    event = { "BufReadPost" },
}
```

### CI/CD

Add a GitHub Actions workflow to run tests on push:

```yaml
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: rhysd/action-setup-vim@v1
        with:
          neovim: true
          version: stable
      - run: make test
```

### Versioning

Tag releases with semantic versioning. lazy.nvim can pin to specific versions:

```lua
{ "author/my-plugin.nvim", version = "^1.0" }
```

---

## Putting It All Together

Let us trace a complete flow through cc-watcher to see how all these pieces
connect.

**1. Neovim starts.** lazy.nvim loads the plugin. `plugin/cc-watcher.lua`
runs: the guard sets `vim.g.loaded_cc_watcher = true`, and six user commands
are registered.

**2. The user opens a file.** The `BufReadPost` autocommand in `watcher.lua`
fires. It takes a snapshot of the file contents (via `snapshots.lua` using
libuv `fs_open`/`fs_read`/`fs_close`) and starts a `fs_event` watcher on
that file.

**3. An external tool modifies the file.** The `fs_event` callback fires
(wrapped in `vim.schedule_wrap`). It reads the file again via libuv,
compares it to the snapshot's raw bytes, and if different, calls
`mark_changed()`. This fires all registered change callbacks.

**4. The sidebar re-renders.** The sidebar's change callback calls `render()`,
which collects files from both the watcher (live changes) and session JSONL
(historical changes). It builds a list of lines with directory grouping,
creates extmarks for highlighting (icons, indicators, stats), and writes
them to the sidebar buffer.

**5. The user presses `<CR>` on a file.** The buffer-local keymap fires
`open_with_diff()`, which opens the file in the main window and calls
`diff.show()`. This computes hunks using `vim.diff()`, creates extmarks for
added/changed/deleted lines and virtual text, and sets up `]c`/`[c`/`cr`
keymaps for navigation and revert.

**6. The user runs `:ClaudeTrouble`.** The command stub in `plugin/` calls
`_ensure_setup()`, verifies the trouble integration is enabled, requires
trouble.nvim, and opens it with mode `"claude"`. trouble.nvim discovers the
source at `lua/trouble/sources/claude.lua`, which delegates to
`cc-watcher/trouble.lua`. The `get()` function collects files, computes
hunks, creates `Item` objects, and passes them to trouble's callback.

Every concept in this guide -- the module system, configuration, autocommands,
keymaps, buffer management, extmarks, window management, libuv I/O, shell
commands, plugin integration, and testing -- plays a role in this flow.

---

## Quick Reference

| Task | API |
|------|-----|
| Create buffer | `vim.api.nvim_create_buf(listed, scratch)` |
| Set buffer option | `vim.bo[buf].option = value` |
| Set window option | `vim.wo[win].option = value` |
| Get buffer lines | `vim.api.nvim_buf_get_lines(buf, start, end, strict)` |
| Set buffer lines | `vim.api.nvim_buf_set_lines(buf, start, end, strict, lines)` |
| Create extmark | `vim.api.nvim_buf_set_extmark(buf, ns, line, col, opts)` |
| Clear extmarks | `vim.api.nvim_buf_clear_namespace(buf, ns, start, end)` |
| Create namespace | `vim.api.nvim_create_namespace(name)` |
| Set highlight | `vim.api.nvim_set_hl(0, name, opts)` |
| Create autocmd | `vim.api.nvim_create_autocmd(event, opts)` |
| Create augroup | `vim.api.nvim_create_augroup(name, opts)` |
| Create command | `vim.api.nvim_create_user_command(name, fn, opts)` |
| Set keymap | `vim.keymap.set(mode, lhs, rhs, opts)` |
| Delete keymap | `vim.keymap.del(mode, lhs, opts)` |
| Open window | `vim.api.nvim_open_win(buf, enter, config)` |
| Close window | `vim.api.nvim_win_close(win, force)` |
| Read file (libuv) | `vim.uv.fs_open` / `fs_fstat` / `fs_read` / `fs_close` |
| Watch file | `vim.uv.new_fs_event()` then `handle:start(path, {}, cb)` |
| Run shell cmd | `vim.fn.systemlist(cmd)` then check `vim.v.shell_error` |
| Merge tables | `vim.tbl_deep_extend("force", defaults, opts)` |
| Notify user | `vim.notify(msg, level, opts)` |
| Inspect value | `vim.inspect(value)` |
| Schedule safely | `vim.schedule(fn)` or `vim.schedule_wrap(fn)` |
| Delay execution | `vim.defer_fn(fn, ms)` |
| Debounce timer | `vim.uv.new_timer()` then `timer:start(ms, 0, cb)` |

---

*This guide was written using cc-watcher.nvim as a reference implementation.
Explore its source code for complete, working examples of every pattern
described here.*
