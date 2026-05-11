# nvim + Keyboard Optimization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the unused MOUSE layer on the Baseform 36-key keymap with a DEV layer (single-key access to nvim save/quit/hunk-nav/window/jumplist), set up nvim with kickstart + LSPs + 4 plugins tuned for diff review and React+PHP, write a minimal tmux config for worktree-session-per-branch, and document the whole thing across two git repos.

**Architecture:** Two coordinated repos. `baseform36` (existing ZMK firmware) gains a DEV layer in `config/miryoku-colemakdh/baseform.keymap` plus `docs/keymap/` containing standalone HTML visualizations. `nvim-baseform36` (new, at `~/Projects/nvim-baseform36/`, symlinked into `~/.config/nvim`) holds kickstart-based `init.lua`, custom plugins, tmux config, and workflow docs.

**Tech Stack:** ZMK (Zephyr-based keyboard firmware, DTS keymap files, pytest validators), Neovim 0.10+, kickstart.nvim (single-file lua config), lazy.nvim (plugin manager), Mason (LSP/tool installer), tmux 3+, git, GitHub CLI (`gh`).

**Spec reference:** `docs/superpowers/specs/2026-05-09-nvim-keyboard-optimization-design.md`

---

## Prerequisites

Before starting, verify these are installed:

```bash
which nvim       # Neovim 0.10 or newer
which git        # Any modern version
which gh         # GitHub CLI for repo creation
which tmux       # tmux 3+
which fd         # used by telescope (kickstart prefers it)
which rg         # ripgrep, used by telescope grep
which node       # for LSP servers
```

If any are missing on macOS:
```bash
brew install neovim git gh tmux fd ripgrep node
```

## File Structure

**`baseform36` repo (existing, modifications + additions):**

```
config/miryoku-colemakdh/baseform.keymap        # MODIFY: layer rename, macros, dev_layer body
docs/superpowers/specs/2026-05-09-...md         # ALREADY EXISTS: design doc
docs/superpowers/plans/2026-05-11-...md         # ALREADY EXISTS: this file
docs/keymap/                                    # CREATE: visualization directory
docs/keymap/README.md                           # CREATE: markdown index
docs/keymap/01-base.html                        # CREATE: base layer viz
docs/keymap/02-dev.html                         # CREATE: dev layer viz (the new one)
docs/keymap/03-nav.html                         # CREATE: nav layer viz
docs/keymap/04-sym.html                         # CREATE: sym layer viz
docs/keymap/05-num.html                         # CREATE: num layer viz
README.md                                       # MODIFY: link to docs/keymap and nvim repo
```

**`nvim-baseform36` repo (new at `~/Projects/nvim-baseform36/`):**

```
init.lua                                        # CREATE: kickstart-derived
lua/custom/plugins/diffview.lua                 # CREATE
lua/custom/plugins/fugitive.lua                 # CREATE
lua/custom/plugins/autotag.lua                  # CREATE
lua/custom/plugins/oil.lua                      # CREATE
tmux.conf                                       # CREATE: multiplexer config
README.md                                       # CREATE: install + overview
docs/workflow.md                                # CREATE: agentic-nvim cycle
docs/keymap-reference.md                        # CREATE: links to baseform36 keymap docs
docs/lsp-setup.md                               # CREATE: Mason commands
docs/plugins.md                                 # CREATE: plugin rationale
docs/leader-bindings.md                         # CREATE: leader cheat-sheet
docs/tmux.md                                    # CREATE: session-per-worktree pattern
.gitignore                                      # CREATE
```

---

## Task 1: Firmware — Add macros block to baseform.keymap

**Files:**
- Modify: `config/miryoku-colemakdh/baseform.keymap`

- [ ] **Step 1: Insert macros block after combos, before keymap**

Open `config/miryoku-colemakdh/baseform.keymap`. Find the closing `};` of the `combos {` block (just before `keymap {` opens). Insert this block immediately after the combos closing brace and before `keymap {`:

```dts
  macros {
    m_save: m_save {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp COLON &kp W &kp RET>;
    };
    m_quit: m_quit {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp COLON &kp Q &kp RET>;
    };
    m_next_hunk: m_next_hunk {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp RBKT &kp C>;
    };
    m_prev_hunk: m_prev_hunk {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp LBKT &kp C>;
    };
    m_next_qf: m_next_qf {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp RBKT &kp Q>;
    };
    m_prev_qf: m_prev_qf {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp LBKT &kp Q>;
    };
    m_win_left: m_win_left {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp LC(W) &kp H>;
    };
    m_win_down: m_win_down {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp LC(W) &kp J>;
    };
    m_win_up: m_win_up {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp LC(W) &kp K>;
    };
    m_win_right: m_win_right {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp LC(W) &kp L>;
    };
    m_vsplit: m_vsplit {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp LC(W) &kp V>;
    };
    m_hsplit: m_hsplit {
      compatible = "zmk,behavior-macro";
      #binding-cells = <0>;
      bindings = <&kp LC(W) &kp S>;
    };
  };
```

- [ ] **Step 2: Verify file parses by running pytest**

Run: `cd /Users/davedivinagracia/Projects/baseform36 && pytest -q`

Expected: all tests pass (the existing suite validates Kconfig and build matrix, not layer names).

- [ ] **Step 3: Commit the macros addition**

```bash
git add config/miryoku-colemakdh/baseform.keymap
git commit -m "feat: add ZMK macros for nvim save/quit/hunk-nav/window-mgmt"
```

---

## Task 2: Firmware — Rename MOUSE layer to DEV

**Files:**
- Modify: `config/miryoku-colemakdh/baseform.keymap`

- [ ] **Step 1: Rename the layer #define**

Find line near top of file:
```dts
#define MOUSE 2
```
Change to:
```dts
#define DEV 2
```

- [ ] **Step 2: Update the base-layer thumb binding**

Find line 221 (in `base_layer`):
```dts
&lt_spc MOUSE TAB  &lt_spc NAV SPACE  &lt_spc MEDIA ESC
```
Change `MOUSE` to `DEV`:
```dts
&lt_spc DEV TAB  &lt_spc NAV SPACE  &lt_spc MEDIA ESC
```

- [ ] **Step 3: Rename the layer block from mouse_layer to dev_layer**

Find the block starting `// Layer 2: MOUSE` (around line 242). Replace the comment, the block name, and the display name:

```dts
    // Layer 2: DEV (Right thumb hold - nvim diff/save/window/jumplist) - 54 keys
    dev_layer {
      display-name = "Dev";
```

- [ ] **Step 4: Run pytest to verify rename didn't break anything**

Run: `pytest -q`

Expected: all tests pass.

- [ ] **Step 5: Commit the rename**

```bash
git add config/miryoku-colemakdh/baseform.keymap
git commit -m "refactor: rename MOUSE layer to DEV (no binding changes yet)"
```

---

## Task 3: Firmware — Wire DEV layer bindings

**Files:**
- Modify: `config/miryoku-colemakdh/baseform.keymap`

- [ ] **Step 1: Replace the dev_layer bindings**

In the `dev_layer` block (formerly `mouse_layer`), replace the entire `bindings = <...>;` with this body. The left hand gets DEV macros; right hand stays mostly transparent with the existing right-hand mods on home row.

```dts
      bindings = <
        // Number row (12 keys - all none)
        &none  &none  &none  &none  &none  &none     &none  &none  &none  &none  &none  &none
        // Top row: jumplist + quickfix on left; right hand trans
        &none  &kp LC(O)   &kp LC(I)    &m_prev_qf   &m_next_qf   &none           &trans  &trans     &trans     &trans     &trans     &none
        // Home row: save + window movement on left; right home row mods preserved
        &none  &m_save     &m_win_left  &m_win_down  &m_win_up    &m_win_right    &trans  &kp RSHFT  &kp RCTRL  &kp RALT   &kp RGUI   &none
        // Bottom row: quit + splits + hunk nav on left; right hand trans
        &none  &m_quit     &m_vsplit    &m_hsplit    &m_prev_hunk &m_next_hunk    &trans  &trans     &trans     &trans     &trans     &none
        // Thumbs (all transparent)
                                        &trans  &trans  &trans     &trans  &trans  &trans
      >;
```

- [ ] **Step 2: Run pytest to verify**

Run: `pytest -q`

Expected: all tests pass.

- [ ] **Step 3: Commit the DEV layer bindings**

```bash
git add config/miryoku-colemakdh/baseform.keymap
git commit -m "feat: wire DEV layer with save/quit/hunk/window/jumplist macros"
```

- [ ] **Step 4: Push to trigger CI build**

```bash
git push origin main   # or your branch
```

Expected: GitHub Actions workflow `build.yml` runs and produces firmware artifacts.

- [ ] **Step 5: Download CI artifacts and flash**

1. Open GitHub Actions tab → latest workflow run → Artifacts section
2. Download `firmware-miryoku-colemakdh-duo-left` and `firmware-miryoku-colemakdh-duo-right` (or the trio variants — match your physical split)
3. Put each half in bootloader mode (double-tap reset) → drag the `.uf2` file onto the mounted volume
4. Both halves should reboot and reconnect over BT

- [ ] **Step 6: Smoke test the DEV layer**

1. Open any text editor (TextEdit, VS Code, supacode)
2. Hold the right inner thumb (TAB key) for ~200ms
3. Tap the left pinky on the home row (the `A` position)
4. Expected output: `:w` followed by a newline (i.e., the macro fired)
5. Hold TAB → tap left pinky on bottom row (Z position) → expected: `:q` + newline
6. Hold TAB → tap V (bottom-rightmost left-hand position) → expected: `]c` typed

If any of these fail, recheck the keymap edits or re-flash.

---

## Task 4: Initialize nvim-baseform36 repo

**Files:**
- Create: `~/Projects/nvim-baseform36/` (new repo)
- Symlink: `~/.config/nvim` → `~/Projects/nvim-baseform36`

- [ ] **Step 1: Back up any existing nvim config**

```bash
if [ -e ~/.config/nvim ]; then
  mv ~/.config/nvim ~/.config/nvim.bak.$(date +%Y%m%d-%H%M%S)
fi
```

Expected: either no output (nothing to back up) or the existing config is renamed.

- [ ] **Step 2: Clone kickstart.nvim into the project location**

```bash
mkdir -p ~/Projects
git clone https://github.com/nvim-lua/kickstart.nvim.git ~/Projects/nvim-baseform36
cd ~/Projects/nvim-baseform36
```

- [ ] **Step 3: Detach from kickstart's git history and start fresh**

```bash
rm -rf .git
git init
git add -A
git commit -m "chore: initial commit from kickstart.nvim template"
```

- [ ] **Step 4: Create the symlink so nvim finds the config**

```bash
ln -s ~/Projects/nvim-baseform36 ~/.config/nvim
ls -la ~/.config/nvim
```

Expected: `ls -la` shows `~/.config/nvim -> /Users/davedivinagracia/Projects/nvim-baseform36`.

- [ ] **Step 5: Launch nvim to bootstrap plugins**

Run: `nvim`

Expected: lazy.nvim auto-installs the kickstart plugins (you'll see a progress UI). When it's done, the home screen appears. Quit with `:q`.

- [ ] **Step 6: Verify the install is clean**

Run: `nvim --headless +qa 2>&1 | head -20`

Expected: no errors. (Some "checking for updates" messages are fine.)

---

## Task 5: Add web stack + PHP LSPs via Mason

**Files:**
- Modify: `~/Projects/nvim-baseform36/init.lua`

- [ ] **Step 1: Find the `servers` table in init.lua**

In `init.lua`, search for `local servers = {`. Kickstart defines several LSPs there (e.g., `lua_ls`). You'll add to this table.

- [ ] **Step 2: Add the web + PHP LSP entries**

Inside the `servers = {` block, add:

```lua
    vtsls = {},
    intelephense = {},
    eslint = {},
    tailwindcss = {},
    cssls = {},
    html = {},
    emmet_language_server = {},
    jsonls = {},
```

- [ ] **Step 3: Find the Mason `ensure_installed` list**

Search `init.lua` for `ensure_installed = vim.tbl_keys`. Kickstart computes Mason's install list from `servers` plus extra tools. Find the line that looks like:

```lua
local ensure_installed = vim.tbl_keys(servers or {})
vim.list_extend(ensure_installed, {
  'stylua',
})
```

- [ ] **Step 4: Add prettierd and php-cs-fixer to the extra tools**

Change the `vim.list_extend(ensure_installed, { ... })` call to include the formatters:

```lua
vim.list_extend(ensure_installed, {
  'stylua',
  'prettierd',
  'php-cs-fixer',
})
```

- [ ] **Step 5: Launch nvim to trigger Mason install**

Run: `nvim`

Then in nvim, run: `:Mason`

Expected: a Mason window opens showing all servers. Wait until each one shows `[Installed]`. This can take 1-3 minutes on first run.

- [ ] **Step 6: Verify by opening a TS file**

Quit nvim, then create a throwaway file:
```bash
echo 'const x: string = 1;' > /tmp/test.tsx
nvim /tmp/test.tsx
```

Expected: after a moment, an LSP diagnostic appears on the line (`Type 'number' is not assignable to type 'string'`). Quit with `:q!`.

- [ ] **Step 7: Commit**

```bash
cd ~/Projects/nvim-baseform36
git add init.lua
git commit -m "feat: add web stack + PHP LSPs (vtsls, intelephense, tailwind, eslint)"
```

---

## Task 6: Add Treesitter parsers for the stack

**Files:**
- Modify: `~/Projects/nvim-baseform36/init.lua`

- [ ] **Step 1: Find the Treesitter `ensure_installed` array**

In `init.lua`, search for `'nvim-treesitter/nvim-treesitter'`. Inside that plugin spec's `opts`, find:

```lua
ensure_installed = { 'bash', 'c', 'diff', 'html', 'lua', ... },
```

- [ ] **Step 2: Add the web + PHP parsers**

Update the array to include:

```lua
ensure_installed = {
  'bash', 'c', 'diff', 'html', 'lua', 'luadoc', 'markdown', 'markdown_inline',
  'query', 'vim', 'vimdoc',
  -- web stack
  'javascript', 'typescript', 'tsx', 'css', 'scss', 'json', 'jsonc', 'yaml',
  -- PHP backend
  'php', 'phpdoc',
},
```

Keep the rest of the treesitter `opts` (highlight, indent, etc.) unchanged.

- [ ] **Step 3: Launch nvim and update parsers**

Run: `nvim`

In nvim: `:TSUpdate`

Expected: nvim downloads and compiles the new parsers. Wait for "TSUpdate done".

- [ ] **Step 4: Verify by opening a TSX file**

```bash
echo '<div className="bg-blue">hello</div>' > /tmp/test.tsx
nvim /tmp/test.tsx
```

Expected: syntax highlighting on the JSX. Quit with `:q!`.

- [ ] **Step 5: Commit**

```bash
git add init.lua
git commit -m "feat: add treesitter parsers for TS/TSX/PHP/CSS"
```

---

## Task 7: Enable format-on-save with conform.nvim

**Files:**
- Modify: `~/Projects/nvim-baseform36/init.lua`

- [ ] **Step 1: Find the conform.nvim plugin block**

Search `init.lua` for `'stevearc/conform.nvim'`. Kickstart ships conform as an optional plugin behind a commented-out block, or sometimes inside `kickstart-plugins` as an `import`. If you see a `-- { import = 'kickstart.plugins.lint' }` style line, your version may have conform ready; if there's a `disabled` flag or commented block, you'll enable it.

- [ ] **Step 2: Configure formatters_by_ft**

Inside conform's `opts` table, set:

```lua
formatters_by_ft = {
  lua = { 'stylua' },
  javascript      = { 'prettierd', 'prettier', stop_after_first = true },
  javascriptreact = { 'prettierd', 'prettier', stop_after_first = true },
  typescript      = { 'prettierd', 'prettier', stop_after_first = true },
  typescriptreact = { 'prettierd', 'prettier', stop_after_first = true },
  html = { 'prettierd' },
  css  = { 'prettierd' },
  scss = { 'prettierd' },
  json = { 'prettierd' },
  jsonc = { 'prettierd' },
  yaml = { 'prettierd' },
  markdown = { 'prettierd' },
  php = { 'php_cs_fixer' },
},
```

- [ ] **Step 3: Enable format-on-save**

In the same `opts` table, set:

```lua
format_on_save = function(bufnr)
  local disable_filetypes = { c = true, cpp = true }
  local lsp_format_opt = disable_filetypes[vim.bo[bufnr].filetype] and 'never' or 'fallback'
  return {
    timeout_ms = 500,
    lsp_format = lsp_format_opt,
  }
end,
```

(Kickstart often ships this exact block — just verify it's not commented out.)

- [ ] **Step 4: Verify format-on-save**

```bash
echo 'const x={a:1,b:2}' > /tmp/test.ts
nvim /tmp/test.ts
```

In nvim, save with `:w`. The file should reformat to:
```ts
const x = { a: 1, b: 2 };
```
Quit with `:q!`.

- [ ] **Step 5: Commit**

```bash
git add init.lua
git commit -m "feat: enable format-on-save for web stack + PHP"
```

---

## Task 8: Add diffview.nvim plugin

**Files:**
- Create: `~/Projects/nvim-baseform36/lua/custom/plugins/diffview.lua`

- [ ] **Step 1: Ensure kickstart's custom plugin import is enabled**

In `init.lua`, search for `kickstart/plugins`. Near the bottom you should see:

```lua
-- { import = 'custom.plugins' },
```

Uncomment the line so it reads:

```lua
{ import = 'custom.plugins' },
```

(If your version has it active already, skip this step.)

- [ ] **Step 2: Create the diffview plugin file**

```bash
mkdir -p ~/Projects/nvim-baseform36/lua/custom/plugins
```

Create file `~/Projects/nvim-baseform36/lua/custom/plugins/diffview.lua` with:

```lua
return {
  'sindrets/diffview.nvim',
  cmd = { 'DiffviewOpen', 'DiffviewClose', 'DiffviewToggleFiles', 'DiffviewFocusFiles' },
  keys = {
    { '<leader>gd', '<cmd>DiffviewOpen<cr>', desc = '[G]it [D]iff view' },
    { '<leader>gD', '<cmd>DiffviewClose<cr>', desc = '[G]it [D]iff close' },
    { '<leader>gh', '<cmd>DiffviewFileHistory %<cr>', desc = '[G]it file [H]istory' },
  },
  opts = {},
}
```

- [ ] **Step 3: Launch nvim and let lazy install diffview**

Run: `nvim`

Expected: lazy.nvim detects the new plugin and installs it. You may see a "Restart neovim" prompt — quit (`:q`) and relaunch.

- [ ] **Step 4: Test :DiffviewOpen in a git repo**

```bash
cd ~/Projects/baseform36   # any repo with commits
nvim
```

In nvim: `:DiffviewOpen`

Expected: a side-by-side diff opens showing changes vs HEAD. The left pane is the original; the right is the working tree. Quit with `:DiffviewClose` or `:q`.

- [ ] **Step 5: Test the leader binding**

In nvim inside a git repo, tap `Space` then `g` then `d`. Expected: diffview opens.

- [ ] **Step 6: Commit**

```bash
cd ~/Projects/nvim-baseform36
git add init.lua lua/custom/plugins/diffview.lua
git commit -m "feat: add diffview.nvim for side-by-side git diff"
```

---

## Task 9: Add nvim-ts-autotag plugin

**Files:**
- Create: `~/Projects/nvim-baseform36/lua/custom/plugins/autotag.lua`

- [ ] **Step 1: Create the autotag plugin file**

Create `~/Projects/nvim-baseform36/lua/custom/plugins/autotag.lua`:

```lua
return {
  'windwp/nvim-ts-autotag',
  event = 'InsertEnter',
  opts = {},
}
```

- [ ] **Step 2: Launch nvim to install**

Run: `nvim`

Expected: lazy installs the plugin.

- [ ] **Step 3: Test JSX tag autoclose**

```bash
echo '' > /tmp/test.tsx
nvim /tmp/test.tsx
```

In nvim:
1. Press `i` to enter insert mode
2. Type `<div>` (literal characters)
3. Expected: the editor immediately inserts the closing `</div>` after the cursor

Quit with `:q!`.

- [ ] **Step 4: Commit**

```bash
git add lua/custom/plugins/autotag.lua
git commit -m "feat: add nvim-ts-autotag for JSX/HTML tag autoclose"
```

---

## Task 10: Add vim-fugitive plugin

**Files:**
- Create: `~/Projects/nvim-baseform36/lua/custom/plugins/fugitive.lua`

- [ ] **Step 1: Create the fugitive plugin file**

Create `~/Projects/nvim-baseform36/lua/custom/plugins/fugitive.lua`:

```lua
return {
  'tpope/vim-fugitive',
  cmd = { 'Git', 'G', 'Gdiffsplit', 'Gvdiffsplit', 'Gblame', 'Glog' },
  keys = {
    { '<leader>gs', '<cmd>Git<cr>', desc = '[G]it [S]tatus' },
    { '<leader>gb', '<cmd>Git blame<cr>', desc = '[G]it [B]lame' },
    { '<leader>gl', '<cmd>Git log --oneline<cr>', desc = '[G]it [L]og' },
  },
}
```

- [ ] **Step 2: Launch nvim to install**

Run: `nvim`

- [ ] **Step 3: Test :Git status**

In a git repo:
```bash
cd ~/Projects/baseform36
nvim
```

In nvim, run `:Git`. Expected: a fugitive status buffer opens showing tracked changes.

- [ ] **Step 4: Commit**

```bash
cd ~/Projects/nvim-baseform36
git add lua/custom/plugins/fugitive.lua
git commit -m "feat: add vim-fugitive for git operations in nvim"
```

---

## Task 11: Add oil.nvim plugin

**Files:**
- Create: `~/Projects/nvim-baseform36/lua/custom/plugins/oil.lua`

- [ ] **Step 1: Create the oil plugin file**

Create `~/Projects/nvim-baseform36/lua/custom/plugins/oil.lua`:

```lua
return {
  'stevearc/oil.nvim',
  dependencies = { 'nvim-tree/nvim-web-devicons' },
  keys = {
    { '<leader>e', '<cmd>Oil<cr>', desc = 'File [E]xplorer (oil)' },
    { '-', '<cmd>Oil<cr>', desc = 'Open parent dir in oil' },
  },
  opts = {
    default_file_explorer = true,
    view_options = { show_hidden = true },
  },
}
```

- [ ] **Step 2: Launch nvim to install**

Run: `nvim`

- [ ] **Step 3: Test oil**

In nvim, press `Space` then `e`. Expected: a buffer opens showing the current directory's contents as editable text. Navigate with arrows. Press `Enter` on a file to open it. Press `-` to go to parent. Quit oil with `:q`.

- [ ] **Step 4: Commit**

```bash
git add lua/custom/plugins/oil.lua
git commit -m "feat: add oil.nvim for buffer-style file management"
```

---

## Task 12: Write the tmux configuration

**Files:**
- Create: `~/Projects/nvim-baseform36/tmux.conf`
- Symlink: `~/.tmux.conf` → `~/Projects/nvim-baseform36/tmux.conf`

- [ ] **Step 1: Create tmux.conf**

Create `~/Projects/nvim-baseform36/tmux.conf`:

```tmux
# True color support
set -g default-terminal "tmux-256color"
set -ag terminal-overrides ",*:RGB"

# nvim-friendly settings
set -g escape-time 0          # zero-latency ESC for nvim mode switches
set -g focus-events on        # nvim auto-reload on external file change

# Quality of life
set -g mouse on
set -g history-limit 100000
set -g base-index 1
setw -g pane-base-index 1

# Worktree session helper: prefix + W prompts for a name and creates a session at the current pane's cwd
bind W command-prompt -p "worktree session:" "new-session -d -s '%%' -c '#{pane_current_path}'"

# Easier session switcher: prefix + S
bind S choose-session
```

- [ ] **Step 2: Back up any existing ~/.tmux.conf**

```bash
if [ -e ~/.tmux.conf ] && [ ! -L ~/.tmux.conf ]; then
  mv ~/.tmux.conf ~/.tmux.conf.bak.$(date +%Y%m%d-%H%M%S)
fi
```

- [ ] **Step 3: Symlink the new config**

```bash
ln -s ~/Projects/nvim-baseform36/tmux.conf ~/.tmux.conf
ls -la ~/.tmux.conf
```

Expected: symlink points to the repo file.

- [ ] **Step 4: Reload tmux if a session is running, otherwise smoke test**

If you have a tmux session: `tmux source-file ~/.tmux.conf`

Otherwise smoke test by creating a session:
```bash
tmux new -s test -d -c ~/Projects/baseform36
tmux switch -t test 2>/dev/null || tmux attach -t test
```

Then inside tmux, press `Ctrl-b` then `W`. Expected: a `worktree session:` prompt appears. Type a name and Enter — a new session is created. Detach with `Ctrl-b` `d`.

- [ ] **Step 5: Commit**

```bash
cd ~/Projects/nvim-baseform36
git add tmux.conf
git commit -m "feat: add tmux config for session-per-worktree workflow"
```

---

## Task 13: Write the nvim-baseform36 README

**Files:**
- Create: `~/Projects/nvim-baseform36/README.md`

- [ ] **Step 1: Create README.md**

Create `~/Projects/nvim-baseform36/README.md`:

```markdown
# nvim-baseform36

Neovim + tmux configuration tuned for the [Baseform 36-key keyboard](https://github.com/idvd20/baseform36) running a Miryoku Colemak-DH layout with a custom DEV layer. Built for an agentic coding workflow: Claude Code does the writing, nvim does diff review + small edits, supacode + git worktrees for parallel branches.

## What's in here

| File | Purpose |
|---|---|
| `init.lua` | kickstart.nvim-derived config |
| `lua/custom/plugins/*.lua` | added plugins (diffview, fugitive, autotag, oil) |
| `tmux.conf` | minimal tmux for session-per-worktree |
| `docs/` | workflow, keymap reference, LSP setup, plugin rationale |

## Install

Prerequisites: Neovim 0.10+, git, gh, tmux 3+, fd, ripgrep, node.

```bash
brew install neovim git gh tmux fd ripgrep node   # macOS

# Clone this repo
git clone https://github.com/idvd20/nvim-baseform36.git ~/Projects/nvim-baseform36

# Back up any existing nvim config
mv ~/.config/nvim ~/.config/nvim.bak 2>/dev/null

# Symlink into nvim's expected path
ln -s ~/Projects/nvim-baseform36 ~/.config/nvim

# Symlink tmux config
ln -s ~/Projects/nvim-baseform36/tmux.conf ~/.tmux.conf

# Launch nvim — lazy.nvim auto-installs plugins
nvim
```

First launch installs ~30 plugins via lazy.nvim. Wait for completion, then run `:Mason` and wait for all LSPs and formatters to show `[Installed]`.

## Keyboard

This config assumes the [Baseform 36-key keymap](https://github.com/idvd20/baseform36/tree/main/docs/keymap) with the DEV layer (right inner thumb hold → left hand becomes nvim action keys).

See `docs/keymap-reference.md` for the chord cheat-sheet.

## Documentation

- [Workflow](docs/workflow.md) — the agentic-nvim cycle, end-to-end
- [Keymap reference](docs/keymap-reference.md) — which key triggers what
- [LSP setup](docs/lsp-setup.md) — Mason install + per-language details
- [Plugins](docs/plugins.md) — what each plugin does and why it earned its slot
- [Leader bindings](docs/leader-bindings.md) — `<leader>` cheat-sheet
- [tmux](docs/tmux.md) — session-per-worktree pattern

## License

MIT.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with install instructions and overview"
```

---

## Task 14: Write the nvim-baseform36 doc files

**Files:**
- Create: `~/Projects/nvim-baseform36/docs/workflow.md`
- Create: `~/Projects/nvim-baseform36/docs/keymap-reference.md`
- Create: `~/Projects/nvim-baseform36/docs/lsp-setup.md`
- Create: `~/Projects/nvim-baseform36/docs/plugins.md`
- Create: `~/Projects/nvim-baseform36/docs/leader-bindings.md`
- Create: `~/Projects/nvim-baseform36/docs/tmux.md`

- [ ] **Step 1: Create docs directory**

```bash
mkdir -p ~/Projects/nvim-baseform36/docs
```

- [ ] **Step 2: Write workflow.md**

Create `~/Projects/nvim-baseform36/docs/workflow.md`:

```markdown
# The Agentic-nvim Workflow

This config is built for a specific workflow: Claude Code writes the code, nvim is for diff review + small edits + search, supacode + git worktrees handle pane management and parallel branches.

## The daily cycle

### 1. Open the file Claude wrote

Tap `Space` (leader), then `f` then `f` — telescope opens a fuzzy file finder. Type a partial filename, press Enter.

Alternative: `Space` `f` `g` — live grep across the repo. Type the text you remember, press Enter on the match.

### 2. Review the diff hunk-by-hunk

Two layers of granularity:

- **By hunk** (changed block) — hold the right inner thumb (TAB → DEV layer), then tap `V` (next hunk) or `D` (prev hunk).
- **By line** — hold the right thumb (SPC → NAV layer), then tap arrow keys (`R/S/T/G` = ←↓↑→).
- **By page** — same NAV layer, tap PGDN/PGUP on bottom row.

For the full diff view: `Space` `g` `d` opens diffview.nvim side-by-side.

### 3. Pull a snippet to share with Claude

In normal mode, press `v` to enter visual mode. Extend selection with NAV-layer arrows. Then chord `A+C` on home row (the Cmd+C combo). The selected text is now in the macOS clipboard, ready to paste into Claude's prompt.

### 4. Quick edit + save

Press `i` to enter insert mode. Type the fix. Press `Esc` (right pinky thumb) to leave insert mode. Hold TAB (DEV layer) → tap `A` (left pinky) — `:w<CR>` fires, file saved.

### 5. Run a tool in another pane

Switch to a supacode pane running tmux. Run `git`, `gh pr create`, `linear issue ...`, or `npm run test:e2e -- --headed` — supacode handles the visual switch.

### 6. Switch to the next worktree

`tmux switch -t <branch-name>` — instant jump to the matching session. Open nvim there, repeat from step 1.

## What you do NOT do

- Reach for the mouse (everything is keyboard)
- Type long commands (leader bindings cover the common ones)
- Memorize HJKL on Colemak-DH (NAV-layer arrows handle movement)
- Hand-write boilerplate (Claude does that)
```

- [ ] **Step 3: Write keymap-reference.md**

Create `~/Projects/nvim-baseform36/docs/keymap-reference.md`:

```markdown
# Keymap Reference

This config is tuned for the [Baseform 36-key Miryoku Colemak-DH layout](https://github.com/idvd20/baseform36/tree/main/docs/keymap). See the source repo for full layer visualizations.

## Layers at a glance

| Layer | Activator | Purpose |
|---|---|---|
| BASE | (always active) | letters + home row mods |
| NAV | hold right thumb (SPC) | arrows, page nav, Cmd combos |
| **DEV** | hold right inner thumb (TAB) | **nvim save/quit/hunk/window/jumplist** |
| SYM | hold left middle thumb (RET) | `:` `;` `{ } [ ]` `* %` |
| NUM | hold left middle thumb (BSPC) | digits + brackets |
| MEDIA / FUN / MOUSE / BUTTON | various | volume, F-keys, BT — rare |

## DEV layer cheat-sheet (most-used)

Hold the right inner thumb (TAB), then tap one left-hand key:

| Key | Action | What nvim does |
|---|---|---|
| `A` (left pinky home) | `:w<CR>` | save file |
| `Z` (left pinky bottom) | `:q<CR>` | quit |
| `V` (left index inner bottom) | `]c` | jump to next changed hunk |
| `D` (left index bottom) | `[c` | jump to prev changed hunk |
| `R/S/T/G` (left home row middle keys) | `<C-w>` + `h/j/k/l` | window left/down/up/right |
| `X` (left ring bottom) | `<C-w>v` | vertical split |
| `C` (left middle bottom) | `<C-w>s` | horizontal split |
| `Q` (left pinky top) | `<C-o>` | jumplist back |
| `W` (left ring top) | `<C-i>` | jumplist forward |
| `F` (left middle top) | `[q` | prev quickfix |
| `P` (left index inner top) | `]q` | next quickfix |

## NAV layer (movement)

Hold right thumb (SPC), then tap on left hand:

| Key | Action |
|---|---|
| `R` | ← |
| `S` | ↓ |
| `T` | ↑ |
| `G` | → |
| `R` row top (W) | ⌘V (paste) |
| `S` row top (F) | ⌘C (copy) |
| (left bottom row) | HOME / PGDN / PGUP / END |
```

- [ ] **Step 4: Write lsp-setup.md**

Create `~/Projects/nvim-baseform36/docs/lsp-setup.md`:

```markdown
# LSP Setup

LSPs are installed via Mason. After the first `nvim` launch, run `:Mason` and wait for everything to install.

## What's installed

| Tool | What for |
|---|---|
| `vtsls` | TypeScript / JavaScript / JSX / TSX language server |
| `intelephense` | PHP language server |
| `eslint-lsp` | ESLint diagnostics inline |
| `tailwindcss-language-server` | Tailwind class IntelliSense |
| `cssls`, `html-lsp`, `emmet-language-server`, `jsonls` | standard web stack |
| `prettierd` | fast prettier daemon (web formatter) |
| `php-cs-fixer` | PHP formatter |
| `stylua` | Lua formatter (for the config itself) |

## Verification

After install, open a TS file:
```bash
echo 'const x: string = 1;' > /tmp/test.tsx
nvim /tmp/test.tsx
```

You should see an inline diagnostic on the line. Save with `:w` — the file reformats via prettierd.

## Per-language notes

- **vtsls** replaces `ts_ls` (better inlay hints, faster, supports TS project references)
- **intelephense** is the best free PHP LSP. A paid license unlocks more features but free tier is sufficient for the workflow.
- **php-cs-fixer** requires a `.php-cs-fixer.dist.php` config in each PHP project. If missing, it falls back to PSR-12 defaults.
```

- [ ] **Step 5: Write plugins.md**

Create `~/Projects/nvim-baseform36/docs/plugins.md`:

```markdown
# Plugins

Beyond what kickstart.nvim ships, this config adds four plugins. Every plugin has to justify its slot — no statusline themes, no debugging, no terminal-in-nvim.

## Added plugins

| Plugin | Why |
|---|---|
| **diffview.nvim** | Best-in-class git diff UX. `:DiffviewOpen` shows full diff with file tree, side-by-side. Core to the workflow — diff review is what nvim is *for* here. |
| **vim-fugitive** | `:G`, `:Gdiffsplit`, `:Gblame`, `:Git log` — the standard for git ops inside nvim. Pairs with diffview. |
| **nvim-ts-autotag** | Auto-closes JSX/TSX/HTML tags. Tiny but daily quality-of-life. |
| **oil.nvim** | Edit the filesystem like a buffer. Better than netrw or neo-tree for keyboard-only file management. |

## Plugins explicitly NOT added

| Plugin | Why not |
|---|---|
| LazyVim | Kickstart is enough; you own every line |
| vim-tmux-navigator | DEV layer covers `<C-w>hjkl` directly |
| flash.nvim / leap.nvim | NAV-layer arrows cover micro-movement |
| toggleterm.nvim | supacode panes do this better |
| neo-tree | oil.nvim is more keyboard-friendly |
| bufferline / lualine | mini.statusline from kickstart is sufficient |
| nvim-dap | not debugging in nvim |
| nvim-colorizer | revisit if Tailwind class soup becomes unreadable |

## Adding more

If you hit friction, add a single plugin file under `lua/custom/plugins/`. Each plugin lives in its own file. Restart nvim and lazy.nvim picks it up automatically.
```

- [ ] **Step 6: Write leader-bindings.md**

Create `~/Projects/nvim-baseform36/docs/leader-bindings.md`:

```markdown
# Leader Bindings

`<leader>` is `<Space>` (right thumb tap). Tap leader, then a letter or two, and nvim runs the command. The which-key plugin will show you the menu after pressing leader.

## All custom leader bindings

| Sequence | Action | Source |
|---|---|---|
| `<leader>ff` | find files (telescope) | kickstart |
| `<leader>fg` | live grep | kickstart |
| `<leader>fb` | buffers | kickstart |
| `<leader>fh` | help tags | kickstart |
| `<leader>fr` | recent files | kickstart |
| `<leader>gd` | open diffview | this config |
| `<leader>gD` | close diffview | this config |
| `<leader>gh` | file history (diffview) | this config |
| `<leader>gs` | git status (fugitive) | this config |
| `<leader>gb` | git blame | this config |
| `<leader>gl` | git log --oneline | this config |
| `<leader>e` | file explorer (oil) | this config |

## Discovery

After tapping `<Space>`, wait ~250ms. which-key pops up a menu showing every available next-key with a description. You don't need to memorize anything upfront.
```

- [ ] **Step 7: Write tmux.md**

Create `~/Projects/nvim-baseform36/docs/tmux.md`:

```markdown
# tmux: Session-Per-Worktree

This config uses tmux **only for session persistence**, not for splits. supacode handles visual splits.

## Pattern

One tmux session per git worktree, named after the branch.

```bash
# Create a session for a worktree
tmux new -s feature-x -d -c ~/Projects/myrepo-worktrees/feature-x

# Switch to it
tmux switch -t feature-x

# Or use the bind: prefix + W to create with a prompt
# (defined in tmux.conf)
```

## Why

- **Persistence**: sessions survive supacode/terminal restart
- **Quick switching**: `tmux switch -t <branch>` is faster than navigating directories
- **Worktree alignment**: each branch's working state (cwd, env, open processes) is self-contained

## Why NOT also use tmux for splits

supacode (and most modern terminals) already handle splits with built-in shortcuts. Duplicating with tmux means learning two systems. Pick one — supacode — and let tmux focus on sessions.

## Bindings (from tmux.conf)

| Binding | Action |
|---|---|
| `prefix W` | create new worktree session with prompt |
| `prefix S` | session switcher |
| `prefix d` | detach (default) |

(`prefix` is `Ctrl-b` unless rebound.)
```

- [ ] **Step 8: Add .gitignore**

Create `~/Projects/nvim-baseform36/.gitignore`:

```
# nvim lazy-lock — keep or ignore based on preference
# Keep it for reproducibility:
# (no entry for lazy-lock.json — track it)

# Build / install artifacts
plugin/packer_compiled.lua
plugin/lazy.lua
.luarc.json
.luarc.jsonc

# OS
.DS_Store
*.swp
*~

# Editor state
.netrwhist
shada/

# Local-only overrides
lua/local/
```

- [ ] **Step 9: Commit all the docs and gitignore**

```bash
cd ~/Projects/nvim-baseform36
git add docs/ .gitignore
git commit -m "docs: workflow, keymap, LSP, plugins, leader bindings, tmux + gitignore"
```

---

## Task 15: Convert keymap visualizations to standalone HTML in baseform36/docs/keymap/

**Files:**
- Create: `docs/keymap/` directory (in baseform36 repo)
- Create: `docs/keymap/README.md`
- Create: `docs/keymap/01-base.html`, `docs/keymap/02-dev.html`, `docs/keymap/03-nav.html`, `docs/keymap/04-sym.html`, `docs/keymap/05-num.html`
- Modify: `README.md` (baseform36 root)

- [ ] **Step 1: Create the keymap docs directory**

```bash
cd /Users/davedivinagracia/Projects/baseform36
mkdir -p docs/keymap
```

- [ ] **Step 2: Locate the brainstorm session HTML source**

The session visualizations are stored at:

```bash
ls ~/Projects/baseform36/.superpowers/brainstorm/*/content/
```

You'll see files like `keymap-v3-dev-on-left.html`, `nvim-keymap-walkthrough.html`. The most recent (`v3`) is the source of truth for the DEV layer.

- [ ] **Step 3: Create `docs/keymap/01-base.html` as a standalone file**

The brainstorm HTML files use a server-injected wrapper. For standalone files, wrap each in a full HTML document with the CSS inlined. Create `docs/keymap/01-base.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Baseform 36 — Base Layer</title>
  <style>
    body { background: #0e0e0e; color: #ddd; font-family: -apple-system, system-ui, sans-serif; margin: 0; padding: 24px; }
    h2, h3 { color: #f4f4f4; }
    .subtitle { color: #999; }
    .layer { max-width: 920px; margin: 24px auto; padding: 18px 20px; background: #141414; border: 1px solid #2a2a2a; border-radius: 12px; }
    .kbd { display: flex; gap: 24px; justify-content: center; }
    .half { display: grid; grid-template-columns: repeat(5, 64px); grid-template-rows: repeat(3, 64px); gap: 6px; }
    .key { border: 1px solid #333; border-radius: 8px; background: #1a1a1a; display: flex; flex-direction: column; align-items: center; justify-content: center; color: #ccc; font-size: 14px; font-weight: 600; position: relative; padding: 4px; }
    .key .lbl { font-size: 16px; font-weight: 700; color: #f4f4f4; }
    .key .mod { position: absolute; top: 2px; right: 4px; font-size: 9px; color: #ff7e5f; font-weight: 700; }
    .key.home { background: #232323; border-color: #ff7e5f; }
    .thumbs { display: flex; gap: 28px; justify-content: center; margin-top: 14px; }
    .thumb-cluster { display: flex; gap: 6px; }
    .key.thumb { width: 64px; height: 60px; font-size: 11px; line-height: 1.1; padding: 4px; }
    .key.thumb .tap { color: #f4f4f4; font-size: 14px; font-weight: 700; }
    .key.thumb .hold { color: #6ec1ff; font-size: 9px; margin-top: 2px; letter-spacing: 0.5px; }
    nav { max-width: 920px; margin: 0 auto 18px; font-size: 13px; }
    nav a { color: #6ec1ff; text-decoration: none; margin-right: 16px; }
  </style>
</head>
<body>
  <nav>
    <a href="README.md">← Index</a>
    <a href="01-base.html">Base</a>
    <a href="02-dev.html">DEV</a>
    <a href="03-nav.html">NAV</a>
    <a href="04-sym.html">SYM</a>
    <a href="05-num.html">NUM</a>
  </nav>
  <div class="layer">
    <h2>Base layer — Colemak-DH alphas + GACS home row mods</h2>
    <p class="subtitle">Always active. Home row keys also act as modifiers when held (GUI / ALT / CTRL / SHFT).</p>
    <div class="kbd">
      <div class="half">
        <div class="key"><div class="lbl">Q</div></div>
        <div class="key"><div class="lbl">W</div></div>
        <div class="key"><div class="lbl">F</div></div>
        <div class="key"><div class="lbl">P</div></div>
        <div class="key"><div class="lbl">B</div></div>
        <div class="key home"><span class="mod">GUI</span><div class="lbl">A</div></div>
        <div class="key home"><span class="mod">ALT</span><div class="lbl">R</div></div>
        <div class="key home"><span class="mod">CTRL</span><div class="lbl">S</div></div>
        <div class="key home"><span class="mod">SHFT</span><div class="lbl">T</div></div>
        <div class="key"><div class="lbl">G</div></div>
        <div class="key"><div class="lbl">Z</div></div>
        <div class="key"><div class="lbl">X</div></div>
        <div class="key"><div class="lbl">C</div></div>
        <div class="key"><div class="lbl">D</div></div>
        <div class="key"><div class="lbl">V</div></div>
      </div>
      <div class="half">
        <div class="key"><div class="lbl">J</div></div>
        <div class="key"><div class="lbl">L</div></div>
        <div class="key"><div class="lbl">U</div></div>
        <div class="key"><div class="lbl">Y</div></div>
        <div class="key"><div class="lbl">'</div></div>
        <div class="key"><div class="lbl">M</div></div>
        <div class="key home"><span class="mod">SHFT</span><div class="lbl">N</div></div>
        <div class="key home"><span class="mod">CTRL</span><div class="lbl">E</div></div>
        <div class="key home"><span class="mod">ALT</span><div class="lbl">I</div></div>
        <div class="key home"><span class="mod">GUI</span><div class="lbl">O</div></div>
        <div class="key"><div class="lbl">K</div></div>
        <div class="key"><div class="lbl">H</div></div>
        <div class="key"><div class="lbl">,</div></div>
        <div class="key"><div class="lbl">.</div></div>
        <div class="key"><div class="lbl">/</div></div>
      </div>
    </div>
    <div class="thumbs">
      <div class="thumb-cluster">
        <div class="key thumb"><div class="tap">DEL</div><div class="hold">FUN</div></div>
        <div class="key thumb"><div class="tap">BSPC</div><div class="hold">NUM</div></div>
        <div class="key thumb"><div class="tap">RET</div><div class="hold">SYM</div></div>
      </div>
      <div class="thumb-cluster">
        <div class="key thumb"><div class="tap">TAB</div><div class="hold">DEV</div></div>
        <div class="key thumb"><div class="tap">SPC</div><div class="hold">NAV</div></div>
        <div class="key thumb"><div class="tap">ESC</div><div class="hold">MEDIA</div></div>
      </div>
    </div>
  </div>
</body>
</html>
```

- [ ] **Step 4: Create `docs/keymap/02-dev.html` (the new DEV layer)**

Copy the structure from `01-base.html` (head, body, nav block, layer div) but replace the layer content with the DEV mapping. Use this for the layer body:

```html
  <div class="layer">
    <h2>DEV layer — hold right inner thumb (TAB)</h2>
    <p class="subtitle">Single-key access to save, quit, hunk-nav, window movement, jumplist, splits.</p>
    <div class="kbd">
      <div class="half">
        <div class="key home"><div class="lbl">⌃o</div><div style="font-size:9px;color:#888;margin-top:3px">jumplist←</div></div>
        <div class="key home"><div class="lbl">⌃i</div><div style="font-size:9px;color:#888;margin-top:3px">jumplist→</div></div>
        <div class="key home"><div class="lbl">[q</div><div style="font-size:9px;color:#888;margin-top:3px">prev qf</div></div>
        <div class="key home"><div class="lbl">]q</div><div style="font-size:9px;color:#888;margin-top:3px">next qf</div></div>
        <div class="key"><div class="lbl">·</div></div>
        <div class="key home"><div class="lbl">:w↵</div><div style="font-size:9px;color:#888;margin-top:3px">SAVE</div></div>
        <div class="key home"><div class="lbl">⌃w h</div><div style="font-size:9px;color:#888;margin-top:3px">win ←</div></div>
        <div class="key home"><div class="lbl">⌃w j</div><div style="font-size:9px;color:#888;margin-top:3px">win ↓</div></div>
        <div class="key home"><div class="lbl">⌃w k</div><div style="font-size:9px;color:#888;margin-top:3px">win ↑</div></div>
        <div class="key home"><div class="lbl">⌃w l</div><div style="font-size:9px;color:#888;margin-top:3px">win →</div></div>
        <div class="key home"><div class="lbl">:q↵</div><div style="font-size:9px;color:#888;margin-top:3px">quit</div></div>
        <div class="key home"><div class="lbl">⌃w v</div><div style="font-size:9px;color:#888;margin-top:3px">vsplit</div></div>
        <div class="key home"><div class="lbl">⌃w s</div><div style="font-size:9px;color:#888;margin-top:3px">hsplit</div></div>
        <div class="key home"><div class="lbl">[c</div><div style="font-size:9px;color:#888;margin-top:3px">prev hunk</div></div>
        <div class="key home"><div class="lbl">]c</div><div style="font-size:9px;color:#888;margin-top:3px">next hunk</div></div>
      </div>
      <div class="half">
        <div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div>
        <div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div>
        <div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div><div class="key"><div class="lbl">·</div></div>
      </div>
    </div>
  </div>
```

(Keep the same `<head>`, `<nav>`, and surrounding structure from 01-base.html, just changing the `<title>` and the `.layer` block.)

- [ ] **Step 5: Create `03-nav.html`, `04-sym.html`, `05-num.html`**

Each follows the same template (head/nav/body shell) with different layer content:

- **03-nav.html**: left hand top row = ⇧⌘Z ⌘V ⌘C ⌘X ⌘Z; home row = CAPS ← ↓ ↑ →; bottom row = INS HOME PGDN PGUP END
- **04-sym.html**: right hand has `:` `$` `%` `^` `+` on home; `{` `&` `*` `(` `}` on top; `~` `!` `@` `#` `|` on bottom
- **05-num.html**: right hand has digits 1-9 + 0, with `[` `]` on top edges, `=` and `\` on bottom edges

Copy the brainstorm session HTML for guidance (`~/Projects/baseform36/.superpowers/brainstorm/*/content/keymap-v3-dev-on-left.html`).

- [ ] **Step 6: Write `docs/keymap/README.md`**

Create `docs/keymap/README.md`:

```markdown
# Baseform 36 — Keymap Visualizations

Open any of the HTML files directly in your browser — they're self-contained, no server needed.

## Layers

| Layer | When it activates | What it does | Visualization |
|---|---|---|---|
| Base | always | letters, home row mods (GUI/ALT/CTRL/SHFT) | [01-base.html](01-base.html) |
| **DEV** | hold right inner thumb (TAB) | nvim save / quit / hunk-nav / window movement / jumplist | [02-dev.html](02-dev.html) |
| NAV | hold right thumb (SPC) | arrows (← ↓ ↑ →), PgUp/PgDn, Cmd combos | [03-nav.html](03-nav.html) |
| SYM | hold left middle thumb (RET) | `:` `;` `{ } [ ]` `* %` and friends | [04-sym.html](04-sym.html) |
| NUM | hold left middle thumb (BSPC) | digits 0-9, brackets, `=` `\` | [05-num.html](05-num.html) |

## Reading the diagrams

- **Orange-bordered keys** = home row mods or layer-specific action keys
- **Blue text on thumb keys** = layer name when held (e.g., "NAV", "DEV")
- **Top text on thumb keys** = tap action (e.g., "SPC", "TAB")

## Workflow paired with this layout

See [`idvd20/nvim-baseform36`](https://github.com/idvd20/nvim-baseform36) — the Neovim + tmux config built specifically for this keyboard.
```

- [ ] **Step 7: Update root README to link to keymap docs**

Open `/Users/davedivinagracia/Projects/baseform36/README.md`. Add a section near the top (after the project overview):

```markdown
## Keymap visualizations

See [`docs/keymap/`](docs/keymap/README.md) for layer-by-layer visualizations of the Miryoku Colemak-DH layout with the custom DEV layer.

## Paired nvim config

The companion repo [`idvd20/nvim-baseform36`](https://github.com/idvd20/nvim-baseform36) ships a Neovim + tmux setup tuned for this keymap.
```

(If the section names or anchors don't fit your existing README, adapt the wording. The two links are the essentials.)

- [ ] **Step 8: Commit the keymap docs**

```bash
cd /Users/davedivinagracia/Projects/baseform36
git add docs/keymap/ README.md
git commit -m "docs: add keymap layer visualizations + link to nvim-baseform36"
```

---

## Task 16: Publish nvim-baseform36 to GitHub

**Files:**
- None modified (publishing the repo)

- [ ] **Step 1: Verify `gh` is authenticated**

```bash
gh auth status
```

Expected: shows authenticated user. If not, run `gh auth login`.

- [ ] **Step 2: Create the GitHub repo and push**

```bash
cd ~/Projects/nvim-baseform36
gh repo create nvim-baseform36 --public --source=. --push --description "Neovim + tmux config for the Baseform 36-key keyboard with custom DEV layer"
```

Expected: a new public repo at `https://github.com/<your-username>/nvim-baseform36` with all commits pushed.

- [ ] **Step 3: Verify online**

```bash
gh repo view --web
```

Expected: the repo opens in your browser. README renders, all files visible, commit history shows your work.

- [ ] **Step 4: Update baseform36's README link if your username differs from `idvd20`**

If your GitHub username is not `idvd20`, find/replace the URL in `/Users/davedivinagracia/Projects/baseform36/README.md` and `/Users/davedivinagracia/Projects/baseform36/docs/keymap/README.md` to match. Commit:

```bash
cd /Users/davedivinagracia/Projects/baseform36
git add README.md docs/keymap/README.md
git commit -m "docs: correct nvim-baseform36 repo link"
```

---

## Final verification — end-to-end smoke test

After all tasks, run through the full workflow once to confirm everything connects:

- [ ] **Step 1: Open nvim in the baseform36 repo**

```bash
cd ~/Projects/baseform36
nvim
```

Expected: nvim launches without errors.

- [ ] **Step 2: Find a file with telescope**

Tap `Space` → `f` → `f`. A picker opens. Type `keymap` and press Enter. Expected: opens `config/miryoku-colemakdh/baseform.keymap`.

- [ ] **Step 3: Trigger DEV-layer save macro**

Hold right inner thumb (TAB) → tap left pinky home (A). Expected: `:w<CR>` fires (nothing visible if no changes — that's correct).

- [ ] **Step 4: Open diffview**

Tap `Space` → `g` → `d`. Expected: diffview side-by-side opens.

- [ ] **Step 5: Navigate to next hunk via DEV layer**

In the diffview right pane, hold TAB → tap V. Expected: cursor jumps to next changed hunk.

- [ ] **Step 6: Quit via DEV-layer macro**

Hold TAB → tap Z. Expected: nvim quits.

If all six steps work, the setup is complete.

---

## Self-Review (completed)

**Spec coverage:**
- D1 (NAV-layer arrows) — preserved in firmware; no implementation needed beyond existing keymap (Task 2 keeps NAV unchanged)
- D2 (kickstart) — Task 4 clones kickstart
- D3 (MOUSE→DEV) — Tasks 1-3 implement firmware change
- D4 (tmux session-per-worktree) — Task 12
- D5 (4 plugins) — Tasks 8-11 (diffview, fugitive, autotag, oil)
- D6 (two repos + docs) — Tasks 4 (new repo init), 13 (README), 14 (docs), 15 (baseform36 keymap docs), 16 (push)

**Placeholder scan:** No TBD/TODO. All steps have concrete commands or code. The base.html template Step 3 has full content; the nav/sym/num HTML files in Step 5 follow the same template pattern with explicit content for each layer.

**Type consistency:** Layer name `DEV` consistent across firmware Tasks 1-3. Macro names (`m_save`, `m_quit`, etc.) consistent between Task 1 (definition) and Task 3 (usage). Plugin file paths consistent.

**Manual-vs-automated:** Tasks 3 (steps 4-6: build/flash), 4 (manual nvim launch), 5-11 (manual nvim runs for verification) require interactive steps. These are clearly marked as user-driven actions, not commands to script.
