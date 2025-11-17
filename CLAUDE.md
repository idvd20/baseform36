# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a ZMK (Zephyr Mechanical Keyboard) firmware configuration for the Baseform split keyboard. The keyboard supports multiple layouts (QWERTY, Colemak, Dvorak) and two split configurations:

- **Duo**: 2-piece split with left half as central
- **Trio**: 3-piece split with separate central module, left peripheral, and right peripheral

The firmware is built on the ZMK framework and uses Zephyr RTOS, targeting nice!nano v2 boards.

## Build System

### Running Tests

```bash
# Install test dependencies
pip install --upgrade pip pytest pyyaml

# Run all tests
pytest -q

# Run specific test file
pytest tests/test_builds.py -v
pytest tests/test_kconfig_and_oled.py -v
pytest tests/test_studio_support.py -v
```

Tests validate:
- Build configuration completeness (all layout × split combinations exist)
- Kconfig settings for OLED, split roles, and keyboard metadata
- ZMK Studio support configuration

### Building Firmware

Builds are automated via GitHub Actions (`.github/workflows/build.yml`). The workflow:
1. Runs tests first
2. Builds firmware using ZMK's official build workflow
3. Packages artifacts by layout and split type
4. Creates releases and updates README links (on manual dispatch)

Build configuration is in `build.yaml`, which defines:
- Board: `nice_nano_v2`
- Shields: Various combinations for duo/trio centrals and peripherals
- Snippets: `studio-rpc-usb-uart` for centrals (enables ZMK Studio)
- Layout-specific cmake args: `-DZMK_CONFIG=/tmp/zmk-config/config/{layout}`

To trigger a build manually:
- Push to any branch (creates CI artifact with commit hash)
- Use GitHub workflow dispatch with version tag (creates release)

## Repository Structure

### Core Configuration Files

- `build.yaml` - Defines all build targets (layouts × splits)
- `config/{layout}/baseform.keymap` - Layout-specific keymap definitions (qwerty, colemak, dvorak, test)
- `config/shared_layers.dtsi` - Shared layers used across all layouts (symbols, navigation, function, meta)
- `config/layer_defs.dtsi` - Layer number definitions (SYM=1, NAV=2, FUN=3, MET=4)
- `config/keymap_edges.dtsi` - Macros for row/key definitions (ROW1-4, THUMB_ROW, etc.)

### Shield Definitions

Located in `boards/shields/baseform/`:
- `baseform.dtsi` - Base shield definition with matrix transforms, physical layouts, and OLED config
- `baseform_duo_left_central.*` - Duo split left central configuration
- `baseform_trio_base_central.*` - Trio split central module configuration
- `baseform_trio_left_peripheral.*` - Trio split left peripheral configuration
- `baseform_any_right_peripheral.*` - Universal right peripheral (works with both duo/trio)
- `Kconfig.defconfig` - Kconfig defaults for split roles, display, and keyboard name
- `Kconfig.shield` - Shield definitions

### Physical Layout Support

The keyboard supports three physical layouts defined in `baseform.dtsi`:
- `baseform_6x4_lo` - 54-key layout (BF3)
- `baseform_6x3_lo` - 42-key layout (BF2)
- `baseform_5x3_lo` - 36-key layout (BF1)

Each has corresponding matrix transforms (`baseform_6x4_tf`, etc.) and position maps for ZMK Studio compatibility.

## Keymap Architecture

### Layer System

All layouts use a 5-layer system defined in `layer_defs.dtsi`:
0. Base layer (QWERTY/Colemak/Dvorak - layout-specific)
1. Symbols layer (SYM) - Special characters and brackets
2. Navigation layer (NAV) - Numbers, arrows, home/end, page up/down
3. Function layer (FUN) - F-keys, brightness, media controls, lock keys
4. Meta layer (MET) - Bluetooth, power, bootloader, system reset

### Keymap Macros

The `keymap_edges.dtsi` file defines macros for consistent key placement:

- `ROW4` - Top row with numbers (ESC, 1-0, BSPC)
- `ROW3(...)` - Second row with CAPS and DEL flanks
- `ROW2(...)` - Third row with TAB and RET flanks
- `ROW1(...)` - Bottom row with LSHFT and RSHFT flanks
- `THUMB_ROW` - 6 thumb keys with layer taps and hold-taps
- `TRANS_ROW` - All transparent keys (for layers)
- `TRANS_FLANK(...)` - Transparent edge keys with custom center

This macro system allows base layers to focus on alpha keys while edge keys remain consistent.

### Shared Layers

`shared_layers.dtsi` is included by all layout keymaps and defines:
- symbols, navigation, function, and meta layers
- These layers are identical across QWERTY, Colemak, and Dvorak

## Important Configuration Details

### Split Configuration

- **Central halves** (duo left, trio base) set:
  - `ZMK_SPLIT_ROLE_CENTRAL=y`
  - `studio-rpc-usb-uart` snippet (enables ZMK Studio over USB/UART)
  - Layout-specific keymap via cmake args

- **Peripheral halves** (trio left, any right) set:
  - `ZMK_SPLIT=y` only (not central)
  - No layout-specific config (follows central)

### OLED Display

- Only enabled on central halves via `.conf` files
- Uses SSD1306 128x64 OLED over I2C
- Configuration in `baseform.dtsi` at address `0x3c`
- Kconfig automatically enables I2C and SSD1306 driver when `ZMK_DISPLAY=y`

### ZMK Studio Support

ZMK Studio allows runtime keymap editing. Support requires:
- Physical layout definitions (all three defined)
- Position maps (`layouts_baseform_pm`)
- `studio-rpc-usb-uart` snippet in build config (centrals only)
- `zmk,studio-rpc-uart = &uart1` in chosen node

Tests in `test_studio_support.py` validate Studio configuration.

## Making Changes

### Adding a New Layout

1. Create `config/{new_layout}/` directory
2. Add `baseform.keymap`, `baseform.conf`, and `west.yml`
3. Add build entries to `build.yaml` for duo and trio splits
4. Add layout to `layouts` list in `tests/test_builds.py`
5. Run tests to verify

### Modifying Shared Layers

Edit `config/shared_layers.dtsi` - changes apply to all layouts. Be careful with:
- Layer numbering (must match `layer_defs.dtsi`)
- Macro usage (TRANS_ROW, TRANS_FLANK)
- ZMK behavior syntax

### Changing Physical Layout

Edit matrix transforms and physical layout definitions in `boards/shields/baseform/baseform.dtsi`. Note:
- Matrix transform maps physical key positions to matrix rows/columns
- Physical layouts define key dimensions and positions for Studio
- Position maps correlate layout positions to keymap indices

### Modifying Edge Key Behavior

Edit `config/keymap_edges.dtsi` macros to change:
- Number row behavior (ROW4)
- Edge key assignments (CAPS, TAB, SHIFT, etc.)
- Thumb cluster behavior (layer taps, hold-taps)

## Common Issues

### Build Failures

- Ensure all referenced layers exist in keymap (0-4 + extra1-5 reserved)
- Verify shield names match between `build.yaml` and shield files
- Check that cmake-args point to correct config directory

### Test Failures

- `test_builds.py` - Missing build entries in `build.yaml`
- `test_kconfig_and_oled.py` - Kconfig or DTSI syntax/configuration errors
- `test_studio_support.py` - Missing physical layout or position map definitions

### Split Keyboard Issues

- Only centrals have layout-specific keymaps
- Peripherals must pair with correct central
- Right peripheral is universal (works with duo or trio)

## Development Workflow

1. Make changes to keymap or configuration files
2. Run tests locally: `pytest -q`
3. Commit and push to branch (triggers CI build)
4. Download CI artifacts to test firmware
5. Create release via workflow dispatch with version tag (e.g., `v2025.09.18a`)
6. Release workflow automatically updates README.md with download links

## File Naming Conventions

- Layouts: lowercase (qwerty, colemak, dvorak)
- Shields: snake_case with variant suffix (baseform_duo_left_central)
- Artifacts: `{layout}_{split}_split_{position}` (e.g., qwerty_duo_split_left)
- Release zips: `{layout}_{split}-{version}.zip` (e.g., qwerty_duo-v2025.09.18a.zip)
