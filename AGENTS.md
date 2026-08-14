# AGENTS.md

ZMK firmware config for the TOTEM keyboard (XIAO BLE). A personal fork of
[urob/zmk-config](https://github.com/urob/zmk-config); `origin` is the fork,
`urob` is upstream. This is a Devicetree + C-preprocessor project built by
Zephyr's `west` — not a regular language codebase.

## Environment

- Toolchain: Nix flake dev shell + `direnv` (`.envrc` + `flake.nix`) + `just`.
  Run commands **inside the direnv shell** (`direnv allow` once, then re-enter the dir).
  The flake provides `west`, the Zephyr SDK, `python-yq` (golang-yq is rejected by
  `just` recipes), `keymap-drawer`, and `dts-format`.
- Workspace layout: `config/` is your source. `zmk/`, `zephyr/`, `modules/` are
  west-managed checkouts pinned by `config/west.yml` — treat as build inputs, not
  your code. `.build/`, `firmware/`, `draw/*.yaml` are generated and gitignored.
- Pin changes: edit `config/west.yml`, then `just update` (fetches the new pins).
  `just update` **overwrites local edits** in `zmk/` and `modules/` — commit/push
  module changes first. `upgrade-sdk` bumps all nix deps and can break the env.

## Build

- `just build all` or `just build <target>` builds per `build.yaml` (the same
  matrix GitHub Actions uses). `just list` shows valid targets. Output lands in
  `firmware/*.uf2`; build dirs in `.build/<artifact>`.
- Build flags: `xiao_ble//zmk` (note the `//zmk` variant suffix) + `shield: totem_left`/`totem_right`.
- `just flash <target>` flashes via west; `just clean` clears `.build`+`firmware`;
  `just clean-all` also removes `.west` and `zmk/` (requires `just init` to restore).
- Only `config/` and `build.yaml` changes should normally be committed.

## Keymap architecture

- `config/base.keymap` is the shared 34-key layout. `config/totem.keymap` defines
  `ZMK_BASE_LAYER(name, LT, RT, LM, RM, XLB, LB, RB, XRB, XLH, LH, RH, XRH)` with
  physical TOTEM row/thumb order, then `#include "base.keymap"`. The macro arg
  order is load-bearing — match it when adding boards.
- Include order matters in `base.keymap`: `combos.dtsi` must stay after the
  HRM-combo hack (`ZMK_COMBO_8`) defined above it. Layer macros `XXX`/`___`
  (`&none`/`&trans`) and `DEF..MOUSE` are reused across `.dtsi` files.
- Heavy use of helper macros from `zmk-helpers` (`ZMK_HOLD_TAP`, `ZMK_MOD_MORPH`,
  `ZMK_TAP_DANCE`, `ZMK_TRI_STATE`, `ZMK_ADAPTIVE_KEY`) plus behaviors from the
  other modules under `modules/zmk/` (auto-layer, leader-key, tri-state, unicode,
  adaptive-key). These macros hide complex nodes; grep the module source to
  understand what a macro expands to.
- Editing a module's source means committing in that module's own git repo
  (`modules/zmk/<name>/`) and re-pinning in `west.yml`.

## Drawing, formatting, testing

- `just draw` regenerates `draw/*.svg` from `config/base.keymap` (via keymap-drawer + jq).
- `dts-format [--fix] [--use-tabs] [--tab-width N] <files>` is available in the
  shell. **Never run it from the repo root without a file list** — it recurses and
  would reformat the entire `zmk/` and `zephyr/` trees. Guard manually aligned
  keymap blocks with `// dts-format off` / `// dts-format on`.
- `just test <path>` builds a `native_sim//zmk_test_mock` test and diffs keycode
  events against a `keycode_events.snapshot` next to the test config. Use
  `--auto-accept` to update the snapshot after a verified change.

## Conventions

- Keymap and Devicetree files are pure C preprocessor output; `.gitattributes`
  marks `.keymap`/`.dtsi` as C++ for linguist. `.prettierrc` applies to prose only.
- Commits are signed (`git commit -S`).
