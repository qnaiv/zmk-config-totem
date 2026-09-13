# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A [ZMK](https://zmk.dev/) firmware configuration for the TOTEM, a 38-key column-staggered split
keyboard (2x Seeed XIAO BLE). There is no application code here — this repo only holds devicetree
keymap/config files that get compiled into firmware by ZMK's own build system (via GitHub Actions).
Hardware/build-guide repo: https://github.com/GEIGEIGEIST/totem.

## Build / "test" workflow

There is no local build toolchain in this repo and no test suite. Firmware is built entirely by
GitHub Actions:

- `.github/workflows/build.yml` calls ZMK's reusable `build-user-config.yml` workflow.
- `build.yaml` lists the build targets: `board: xiao_ble//zmk` with `shield: totem_left` and
  `shield: totem_right`.
- Push to any branch to trigger a build; download `firmware.zip` from the Actions run, then flash
  `totem_left-...uf2` / `totem_right-...uf2` to each half (double-tap reset to enter the UF2
  bootloader, then drag-and-drop the file onto the mass-storage device that appears).

Before pushing a keymap change, sanity-check every layer still has exactly 38 bindings (one per
physical key — see "Key position numbering" below). A quick way to check:

```bash
python3 - <<'EOF'
import re
content = open("config/totem.keymap").read()
for m in re.finditer(r'(\w+_layer)\s*\{.*?bindings\s*=\s*<(.*?)>;', content, re.S):
    name, body = m.groups()
    print(name, len(re.findall(r'&\S+', body)))
EOF
```

A binding-count mismatch is caught at compile time anyway, but this catches it before pushing.
There is no way to test keymap logic without building and flashing to real hardware.

## Critical: which keymap file is actually live

There are **two** `totem.keymap` / `totem.conf` files in this repo, and only one pair is used:

- `config/totem.keymap` / `config/totem.conf` — **this is the one that gets built.** ZMK's build
  resolves the keymap/conf from the top-level `ZMK_CONFIG` dir (`config/`, per `config/west.yml`'s
  `self.path`) using the shield name with the `_left`/`_right` suffix stripped, which is exactly
  `totem`. This is also the file the repo's `readme.md` tells users to edit.
- `config/boards/shields/totem/totem.keymap` / `.../totem.conf` — the shield module's own bundled
  *default* keymap (Colemak-DH layout, different layer set). It is **not used** by this repo's
  build and has drifted out of sync with the real one. Don't edit it expecting it to do anything;
  don't be confused by its different layout when grepping the repo.

`config/west.yml` pins the `zmk` module to `revision: main` — a moving target, rebuilt fresh on
every push. Be cautious pinning it to an older tag: `build.yaml` already uses the
`xiao_ble//zmk` board-variant suffix, a syntax only understood by ZMK after its Zephyr 4.1 update
(Dec 2025). Pinning to a tag older than that (e.g. `v0.3.0`) makes CMake fail with
`Invalid BOARD; see above.` for both shields.

## Keymap architecture (`config/totem.keymap`)

- Base layer types plain **QWERTY** (not Colemak — that's only in the unused shield-default file
  above). Home row has GACS/mod-tap style mods: `&mt LALT S`, `&mt LCTRL D`, `&mt LSHFT F`, and
  mirrored on the right hand.
- Layers are `#define`d for readability (`BASE 0`, `NAV 1`, `SYM 2`, `ADJ 3`, `TVP1 4`, `TVP2 5`,
  `BT 6`), but the **actual** layer index is the node's position in `keymap { ... }` — the defines
  must stay in sync with that document order if you add/reorder layers.
- `SYM` (layer 2, `sim_layer` node) is the mouse-emulation layer, reached by holding the right
  space/thumb key (`&lt 2 SPACE`). Left-hand E/S/D/F move the cursor, W/R scroll, and the two
  leftmost thumb keys are right-/left-click.
- `BT` (layer 6, `bt_layer` node) holds the Bluetooth profile keys (`BT_CLR`, `BT_SEL 0-3`, at the
  same Z/X/C/V-row positions they used to occupy on `SYM`). It is **not** reached via `&mo`/`&lt`
  on a normal key — it's activated only by the `bt_layer_combo` combo (`key-positions = <34 35>`,
  both space/thumb keys held together), specifically so BT actions can't fire by accident while
  using `SYM` for mouse/symbols.
- Other combos (`lefttab`, `righttab`, `leftesc`) follow the same pattern:
  `key-positions` are indices into the **base layer's `bindings` array, in document order,
  0-indexed** (row-major: row0 keys 0-9, row1 10-19, row2 20-31, thumb row 32-37). This is the
  numbering to use for any new combo.
- Mouse emulation: `&mkp` is buttons only (`LCLK`/`RCLK`); cursor movement needs `&mmv`; scrolling
  needs `&msc`. They're easy to confuse (identical `MOVE_UP`/`MOVE_DOWN`/... style params) but are
  different ZMK behaviors — using the wrong one silently does nothing rather than erroring.
  Movement speed is tuned via `#define ZMK_POINTING_DEFAULT_MOVE_VAL` (must be defined *before*
  `#include <dt-bindings/zmk/mouse.h>`) and the `&mmv { time-to-max-speed-ms = ...; }` override
  near the top of the file. To raise top speed without changing how fast it starts moving, scale
  both numbers by the same factor (speed ramps linearly as `max_speed * elapsed/time_to_max_speed_ms`).
- `CONFIG_ZMK_MOUSE=y` in `config/totem.conf` is a deprecated alias that still cascades to the
  real `CONFIG_ZMK_POINTING`; `CONFIG_ZMK_BLE_PASSKEY_ENTRY=y` was added to fix
  "incorrect PIN/passkey" Bluetooth pairing failures (seen on Windows and Android) by switching
  from Just-Works to passkey-entry pairing.
- `totem_left` is the BLE central (`SHIELD_TOTEM_LEFT` → `ZMK_SPLIT_ROLE_CENTRAL default y` in
  `config/boards/shields/totem/Kconfig.defconfig`); only it pairs with hosts. `totem_right` is the
  peripheral and only talks to the left half.
