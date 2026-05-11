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
