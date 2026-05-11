# nvim + Keyboard Optimization for Agentic Workflow

**Date:** 2026-05-09
**Status:** Design (pending user review → implementation plan)
**Scope:** `baseform36` firmware (single keymap layout) + nvim configuration + tmux config

---

## Context

The user has a 36-key Baseform split keyboard running ZMK firmware with a Miryoku Colemak Mod-DH layout (FLIP thumbs + INVERTED-T arrows on NAV layer, GACS home row mods). The keymap lives in `config/miryoku-colemakdh/baseform.keymap`.

The user wants to optimize the layout for a **terminal-first, agentic coding workflow**:

- **Coding** is done by Claude Code (or similar agent)
- **nvim** is used for **diff review + simple edits** (~70% / 20% / 10% with search)
- **Terminal emulator:** supacode, with **git worktrees** for parallel branches
- **Stack:** React (TS/TSX/JS/JSX) + PHP backend
- **OS:** macOS
- Goal: stay on the keyboard, almost never reach for the mouse

The current layout already includes Cmd-clipboard combos (`A+C/V/X/Z`, `O+C/V/X/Z`) and an INVERTED-T arrow cluster on the NAV layer that mirrors vim's HJKL onto the left hand.

## Goals

1. Make nvim's diff-review loop friction-free: hunk navigation, save, jumplist, window movement should be single-tap or single-chord.
2. Set up nvim with a maintainable, minimal config (kickstart.nvim) that the user owns line-by-line.
3. Wire up LSPs, formatting, and tree-sitter for the user's React + PHP stack.
4. Use tmux only where it adds real value (worktree-aware session persistence), without duplicating supacode's pane management.
5. Avoid plugin bloat — every plugin must justify itself for *this* workflow.
6. **Document the entire setup so it's self-explanatory.** Keymap visualizations, nvim config, and workflow docs all live in version control — both for future reference and so the unique keyboard-firmware↔editor coupling is studyable.

## Non-goals

- Heavy in-buffer text editing optimization (Claude does the writing).
- Replacing supacode panes with tmux panes.
- Configuring nvim debugging (`nvim-dap`) — debugging happens elsewhere.
- Adding HJKL remapping, leap.nvim, flash.nvim, or any motion plugin in the initial setup. NAV-layer arrows cover micro-movement; revisit only if friction emerges.
- Changing base-layer alphas, home row mods, or any layer other than MOUSE.

## Decisions

### D1. Movement strategy — NAV-layer arrows instead of HJKL

The user's NAV layer (held by right thumb / SPC) already has `LEFT/DOWN/UP/RIGHT` on `R/S/T/G` (left home row) — vim's HJKL mirrored to the left hand. nvim accepts arrows natively in every mode, so this is the movement strategy in nvim normal mode too.

**Why:** Colemak-DH scatters HJKL across the right hand. Remapping HJKL fights every plugin that assumes defaults. Arrows on the NAV layer are closer to the home row than HJKL would be on Colemak-DH and work uniformly across nvim, the terminal, and the browser.

**Trade-off accepted:** when navigating, the right thumb stays held — same ergonomic pattern as holding shift while typing capitals. Long-distance jumps will use `w/b/e/f/t/`/`?` natively or be skipped (Claude finds the location).

### D2. Distribution — kickstart.nvim

Use `kickstart.nvim` (single-file, ~600 lines, by tjdevries). Not LazyVim, not NvChad.

**Why:**
- The user owns every line and can debug anything.
- Plugin set under kickstart is identical capability-wise to LazyVim's core (lazy.nvim, telescope, treesitter, mason, conform, gitsigns, which-key, mini.nvim).
- The user is already customizing their keyboard at the firmware layer; same instinct applies to the editor — don't inherit opinions you can't justify.

### D3. Keyboard — repurpose MOUSE layer as DEV layer

The MOUSE layer (held by right inner thumb / TAB) is rarely used in this workflow. Replace it with a DEV layer providing single-key access to nvim's most-used review-and-edit operations.

The DEV layer's action keys live on the **left hand** (the free hand while right thumb is held). The mapping mirrors the NAV-layer arrow ergonomics: home row gets save + window-arrow cluster on the same physical positions where NAV puts cursor arrows.

### D4. Multiplexer — tmux for session persistence only

Use tmux for **session-per-worktree**, not for visual splits. Supacode handles splits. This keeps tmux config minimal and avoids learning two split shortcut systems.

**Why session-per-worktree:** with 5 active worktrees, `tmux switch -t <branch>` is faster than navigating directories. Sessions persist across supacode restarts.

### D5. Plugin selection — minimal, every plugin justified

Beyond what kickstart ships, only four plugins are added: `diffview.nvim`, `vim-fugitive`, `nvim-ts-autotag`, `oil.nvim`. Each one is justified against the workflow below. No statusline themes, no debugging, no terminal-in-nvim, no motion plugins, no buffer-line.

### D6. Documentation strategy — visualizations live with the code

Two repos, both pushable to GitHub:

- **`baseform36`** (existing) — gains `docs/keymap/` containing standalone HTML visualizations of every layer + a markdown index that links to them and explains each layer's purpose. The HTML files open directly in any browser (no server needed). The markdown index renders on GitHub for at-a-glance reference.
- **`nvim-baseform36`** (new) — a separate repo for the nvim configuration. Contains `init.lua` (kickstart-based), custom plugin files, the tmux config, and `docs/` describing the workflow, keymap reference, LSP setup, and install instructions.

The two repos are linked: `nvim-baseform36/README.md` references the keymap visualizations in `baseform36/docs/keymap/`, and `baseform36/README.md` mentions the paired nvim config repo. Anyone landing on either repo can navigate to the other.

**Why two repos:** the keyboard firmware and the editor config evolve on different schedules and target different audiences (ZMK builders vs nvim users). Coupling them in one repo would obscure both.

---

## Architecture

### Firmware (baseform36)

Single file changed: `config/miryoku-colemakdh/baseform.keymap`.

**Layer renumbering:** `#define MOUSE 2` becomes `#define DEV 2`. All other layers unchanged.

**Base-layer thumb:** `&lt_spc MOUSE TAB` becomes `&lt_spc DEV TAB`. Tap behavior (TAB) is unchanged.

**New ZMK macros** (added under `/` alongside `behaviors` and `combos`):

| Macro | Sequence | Purpose |
|---|---|---|
| `m_save` | `: w <CR>` | nvim save |
| `m_quit` | `: q <CR>` | nvim quit |
| `m_next_hunk` / `m_prev_hunk` | `]c` / `[c` | gitsigns/diff hunk nav |
| `m_next_qf` / `m_prev_qf` | `]q` / `[q` | quickfix list nav |
| `m_win_left/down/up/right` | `<C-w>` + `h/j/k/l` | nvim window movement |
| `m_vsplit` / `m_hsplit` | `<C-w>v` / `<C-w>s` | window splits |

**DEV layer binding map** (left hand only; right hand transparent):

```
Top row    [ <C-o>     <C-i>     [q       ]q       —     ]
Home row   [ :w<CR>    <C-w>h    <C-w>j   <C-w>k   <C-w>l ]
Bottom row [ :q<CR>    <C-w>v    <C-w>s   [c       ]c    ]
```

Right home row keeps shifted mods (`RSHFT/RCTRL/RALT/RGUI`) for compatibility with the existing pattern. All other right-hand keys are `&trans` (fall through to base).

**Tests:** existing pytest suite (`test_builds.py`, `test_kconfig_and_oled.py`, `test_studio_support.py`) checks build matrix and shield config — not layer names — so no test changes are needed. Run `pytest -q` after the keymap edit to confirm.

### nvim configuration

**Bootstrap:**
```bash
git clone https://github.com/nvim-lua/kickstart.nvim.git ~/.config/nvim
nvim   # lazy.nvim auto-installs; close/reopen
```

**Mason `ensure_installed` (LSPs + formatters):**
- `vtsls` (TS/JS/JSX/TSX — superior to ts_ls)
- `intelephense` (PHP)
- `eslint-lsp`
- `tailwindcss-language-server`
- `cssls`, `html-lsp`, `emmet-language-server`, `jsonls`
- `prettierd` (web formatter)
- `php-cs-fixer` (PHP formatter)

**Treesitter `ensure_installed` additions:** `typescript`, `tsx`, `javascript`, `html`, `css`, `scss`, `json`, `jsonc`, `php`, `phpdoc`, `yaml`, `markdown`, `markdown_inline`.

**Conform.nvim `formatters_by_ft`:**
```lua
javascript      = { 'prettierd', 'prettier', stop_after_first = true }
javascriptreact = { 'prettierd', 'prettier', stop_after_first = true }
typescript      = { 'prettierd', 'prettier', stop_after_first = true }
typescriptreact = { 'prettierd', 'prettier', stop_after_first = true }
html / css / scss / json / markdown = { 'prettierd' }
php = { 'php_cs_fixer' }
```

**Added plugins** (each in its own file under `~/.config/nvim/lua/custom/plugins/`):

| Plugin | Justification |
|---|---|
| `sindrets/diffview.nvim` | Best-in-class git-diff UX. `:DiffviewOpen` shows full diff with file tree. Core to the workflow. |
| `tpope/vim-fugitive` | `:Gdiffsplit`, `:Gblame`, `:G` for staging/committing. Pairs with diffview for full git ops in nvim. |
| `windwp/nvim-ts-autotag` | JSX/TSX tag auto-close. Daily quality-of-life for React. |
| `stevearc/oil.nvim` | Edit filesystem like a buffer. Better than netrw for keyboard-only file ops. |

**Plugins explicitly NOT added** (with rationale): LazyVim, vim-tmux-navigator (DEV layer covers `<C-w>hjkl`), flash.nvim (NAV-layer arrows cover micro-movement), toggleterm.nvim (supacode panes), neo-tree (oil.nvim is more keyboard-friendly), bufferline / lualine (mini.statusline from kickstart is enough), nvim-dap, nvim-colorizer (revisit if Tailwind class soup becomes unreadable).

### Documentation artifacts

**In `baseform36` repo (additions to existing repo):**

```
docs/
├── keymap/
│   ├── README.md              # markdown index, layer explanations, links to HTML
│   ├── 00-overview.html       # all layers at a glance
│   ├── 01-base.html           # base layer + home row mods
│   ├── 02-dev.html            # NEW dev layer
│   ├── 03-nav.html            # NAV (arrows + page nav)
│   ├── 04-sym.html            # SYM (`:`, brackets, math)
│   ├── 05-num.html            # NUM (digits + brackets)
│   ├── 06-other.html          # MEDIA, FUN, BUTTON
│   └── 07-workflow-cycle.html # the agentic-nvim cycle visualization
└── superpowers/
    └── specs/
        └── 2026-05-09-nvim-keyboard-optimization-design.md  # this file
```

The HTML files are derived from the brainstorming-session visualizations, made standalone (CSS inlined, no server-injected wrapper). They open directly via `file://` URLs and remain functional indefinitely.

The `docs/keymap/README.md` is the entry point: explains what each layer is for, when it activates, and links to the matching HTML. Renders on GitHub for at-a-glance navigation.

**In `nvim-baseform36` repo (new repo, lives at `~/Projects/nvim-baseform36/` and symlinked into `~/.config/nvim`):**

```
nvim-baseform36/
├── README.md                  # install, dependencies, what this is, links to baseform36 keymap
├── init.lua                   # kickstart-derived, with the customizations below
├── lua/
│   └── custom/
│       └── plugins/
│           ├── diffview.lua
│           ├── fugitive.lua
│           ├── autotag.lua
│           └── oil.lua
├── tmux.conf                  # the multiplexer config (or symlinked to ~/.tmux.conf)
├── docs/
│   ├── workflow.md            # the agentic-nvim cycle, narrated
│   ├── keymap-reference.md    # quick reference linking to baseform36/docs/keymap
│   ├── lsp-setup.md           # Mason install commands, LSP server table
│   ├── plugins.md             # what each plugin does and why it earned its slot
│   ├── leader-bindings.md     # `<leader>` key cheat-sheet
│   └── tmux.md                # session-per-worktree pattern
└── .gitignore                 # nvim/lua artifacts
```

The repo lives in `~/Projects/nvim-baseform36/` alongside the `baseform36` repo (consistent project organization). `~/.config/nvim` is a symlink to it, so nvim still finds the config at the conventional path. Edits, commits, and pushes happen from `~/Projects/nvim-baseform36/` like any other project. Pushing to GitHub creates a backup and shareable reference.

### tmux configuration

`~/.tmux.conf` (lives in the `nvim-baseform36` repo as `tmux.conf`; copy or symlink it into `~/.tmux.conf` — either works):
```tmux
set -g default-terminal "tmux-256color"
set -ag terminal-overrides ",*:RGB"
set -g escape-time 0          # critical for nvim ESC responsiveness
set -g focus-events on        # nvim auto-reload on file change
set -g mouse on
bind W command-prompt -p "worktree:" "new-session -d -s '%%' -c '#{pane_current_path}'"
```

Pattern: one `tmux` session per worktree, named after the branch. Switch via `tmux switch -t <branch>`. Supacode owns visual splits; tmux owns session persistence.

---

## Implementation phases

### Phase 1 — Firmware: DEV layer

1. Edit `config/miryoku-colemakdh/baseform.keymap`:
   - Rename `MOUSE` → `DEV` in the `#define` block
   - Update base-layer thumb binding: `&lt_spc DEV TAB`
   - Add ZMK macros block (12 macros listed above)
   - Replace `mouse_layer` body with `dev_layer` mapping
2. Run `pytest -q` to verify tests pass
3. Commit, push, let CI build artifacts
4. Flash both halves; verify in ZMK Studio that the DEV layer name and bindings appear correctly
5. Smoke test in a real nvim session: hold TAB → tap `A` to save → confirm `:w<CR>` fires

### Phase 2 — nvim: kickstart + LSPs

1. Clone kickstart.nvim
2. Read `init.lua` top to bottom (~600 lines)
3. Add Mason `ensure_installed` list
4. Add LSP server configs in the `servers` table
5. Add Treesitter parsers
6. Enable kickstart's bundled `conform.nvim` and configure formatters
7. Test: open a `.tsx` file, save → prettierd runs; type `<div>` → autotag closes (after Phase 3)

### Phase 3 — Diff workflow plugins

1. Add `diffview.nvim`, `vim-fugitive`, `nvim-ts-autotag`, `oil.nvim` (one plugin file each in `lua/custom/plugins/`)
2. Set up leader bindings:
   - `<leader>gd` → `:DiffviewOpen`
   - `<leader>gD` → `:DiffviewClose`
   - `<leader>gs` → `:Git` (fugitive status)
   - `<leader>e` → `:Oil`
3. Verify: `]c` / `[c` jump hunks (gitsigns + DEV-layer macros agree)

### Phase 4 — tmux session-per-worktree

1. Place `tmux.conf` in the `nvim-baseform36` repo
2. Symlink to `~/.tmux.conf`: `ln -s ~/Projects/nvim-baseform36/tmux.conf ~/.tmux.conf`
3. For each active worktree, start a session: `tmux new -s <branch> -c /path/to/worktree`
4. Switch sessions via prefix + `s` or scripted `tmux switch -t <branch>`
5. Verify ESC responsiveness in nvim is unchanged (escape-time 0)

### Phase 5 — Documentation: keymap visualizations + new repo

**Part A — `baseform36/docs/keymap/`:**

1. Create the `docs/keymap/` directory
2. Convert each brainstorming-session HTML into a standalone file (inline the CSS, drop the server-injected helper script reference)
3. Write `docs/keymap/README.md`: brief explanation of each layer + relative links to the HTML files
4. Update `baseform36/README.md` (root) to point to `docs/keymap/` and to the paired `nvim-baseform36` repo

**Part B — `nvim-baseform36` repo (new, at `~/Projects/nvim-baseform36/`):**

1. Back up any existing nvim config: `mv ~/.config/nvim ~/.config/nvim.bak` (if applicable)
2. `mkdir -p ~/Projects/nvim-baseform36 && cd ~/Projects/nvim-baseform36 && git init`
3. Copy the kickstart-derived `init.lua` and the four custom plugin files into the repo
4. Copy `tmux.conf` into the repo
5. Write `README.md` (install steps, prerequisites, links)
6. Write the `docs/` files (`workflow.md`, `keymap-reference.md`, `lsp-setup.md`, `plugins.md`, `leader-bindings.md`, `tmux.md`)
7. Add `.gitignore` for nvim artifacts (lazy-lock.json optional, plugin/, etc.)
8. Initial commit
9. User runs `gh repo create idvd20/nvim-baseform36 --public --source=. --push` to publish (out-of-band of this design)

**Part C — wire the repo into place:**

1. `ln -s ~/Projects/nvim-baseform36 ~/.config/nvim` — nvim reads the config from the symlinked path
2. `ln -s ~/Projects/nvim-baseform36/tmux.conf ~/.tmux.conf` — tmux reads its config from the home dir

The repo lives in `~/Projects/` (consistent with `baseform36` and other work), but nvim and tmux still find their configs where they expect.

---

## The agentic-nvim cycle

1. **Open file Claude wrote:** tap `SPC` (leader) → `ff` (telescope find files), or `fg` for live grep
2. **Review diff hunk-by-hunk:** hold `TAB` (DEV) → tap `V` for `]c` (next hunk), tap `D` for `[c`. Within a hunk, hold `SPC` → arrows scroll line-by-line, PGDN scrolls page
3. **Pull snippet to share with Claude:** tap `v` → select with NAV-layer arrows → Cmd+C combo on home row → into system clipboard
4. **Quick edit + save:** tap `i` → type → ESC → hold `TAB` + tap `A` (single chord fires `:w<CR>`)
5. **Switch to supacode pane:** run `git`, `gh`, `linear`, or `npm run test:e2e -- --headed` — supacode handles the pane switch
6. **Switch to next worktree:** `tmux switch -t <branch>` (or supacode tab) — repeat from step 1

---

## Risks / open questions

1. **`<C-i>` collides with `<Tab>` in terminals.** When the DEV layer fires `<C-i>` for jumplist-forward, it sends the same byte as Tab. nvim distinguishes them in some configs but not all. If this misbehaves, `<C-o>` alone is still useful — `<C-i>` can be replaced with a different binding.
2. **MEDIA layer activates on ESC hold.** Under fast mode-switching, holding ESC past 170ms triggers MEDIA. Not a blocker; can be moved off ESC if it bites.
3. **Supacode pane shortcuts** are unknown; the user will configure them outside this design.
4. **PHP formatter:** `php-cs-fixer` requires a `.php-cs-fixer.dist.php` config in each PHP project. If the user prefers `pretty-php` or `phpcbf`, swap in conform config.

## Future (out of scope for this design)

- Base-layer combos for `:` and `;` if SYM-layer hop is felt
- VIM-helper layer for marks/registers if heavy editing emerges
- Tailwind class-sorting / colorizer if Tailwind grows unreadable
- LSP inlay hints toggling
- AI/Claude integration plugins (when/if available natively in nvim)
